# Reporters & CI Integration

> A test suite only delivers value when it runs reliably in CI and reports results clearly. This is the "how do you wire Playwright into a pipeline at scale" topic.

## TL;DR

- Built-in reporters: `list`/`line` (console), `html` (rich local report), `json`/`junit` (machine-readable), `github` (PR annotations), `blob` (mergeable across shards).
- Use **multiple reporters** at once — e.g. `blob` + `github` in CI, `html` + `list` locally.
- In CI: `npm ci` → `playwright install --with-deps` (cache the browsers) → run (often **sharded**) → **merge blob reports** → upload artifacts.
- Always publish the **HTML report and traces** as artifacts so failures are debuggable after the job ends.
- Gate merges on the suite via branch protection (ties into the Git theme's CI-gating).

## Reporters

```ts
// playwright.config.ts
export default defineConfig({
  reporter: process.env.CI
    ? [['blob'], ['github'], ['junit', { outputFile: 'results.xml' }]]
    : [['html', { open: 'never' }], ['list']],
});
```

| Reporter | Output | Use |
|---|---|---|
| `list` | Detailed per-test console lines. | Local default. |
| `line` | Compact single-line progress. | CI logs where you want brevity. |
| `dot` | One char per test. | Very large suites. |
| `html` | Interactive report (filter, traces, screenshots). | Local + published CI artifact. |
| `json` | Structured results. | Custom dashboards/integrations. |
| `junit` | JUnit XML. | CI test-result UIs (most CI systems parse it). |
| `github` | Inline PR annotations on failing lines. | GitHub Actions. |
| `blob` | Mergeable binary report per shard. | **Sharded CI** — merge into one HTML report. |

`blob` is the key to sharding: each shard emits a blob; you merge them into a single coherent HTML report afterward ([parallelism](./11-parallelism-retries-and-sharding.md)).

## A GitHub Actions workflow (annotated)

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on:
  push: { branches: [main] }
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]      # 4-way sharding across parallel jobs
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci

      # Cache the browser binaries, keyed on the installed Playwright version
      - name: Get Playwright version
        id: pw
        run: echo "v=$(npm ls @playwright/test --json | jq -r '.dependencies["@playwright/test"].version')" >> "$GITHUB_OUTPUT"
      - uses: actions/cache@v4
        id: pw-cache
        with:
          path: ~/.cache/ms-playwright
          key: pw-${{ runner.os }}-${{ steps.pw.outputs.v }}
      - name: Install browsers (+ OS deps)
        run: npx playwright install --with-deps
        if: steps.pw-cache.outputs.cache-hit != 'true'
      - name: Install only OS deps when browsers are cached
        run: npx playwright install-deps
        if: steps.pw-cache.outputs.cache-hit == 'true'

      - name: Run tests (shard ${{ matrix.shard }})
        run: npx playwright test --shard=${{ matrix.shard }}/4
        env:
          CI: true
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_USER: ${{ secrets.TEST_USER }}
          TEST_PASS: ${{ secrets.TEST_PASS }}

      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report/
          retention-days: 7

  merge-report:
    if: ${{ !cancelled() }}
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci
      - uses: actions/download-artifact@v4
        with: { path: all-blob-reports, pattern: blob-report-*, merge-multiple: true }
      - run: npx playwright merge-reports --reporter=html ./all-blob-reports
      - uses: actions/upload-artifact@v4
        with: { name: html-report, path: playwright-report/, retention-days: 14 }
```

### What each part is doing (and why)

- **`reporter: blob` per shard + a `merge-report` job** — sharding produces fragmented results; the merge step recombines them into one HTML report. Without it you'd have four partial reports.
- **Browser caching keyed on the Playwright version** — browsers are large; caching saves minutes per run. The key *must* include the version so an upgrade re-downloads the matching binaries. On a cache hit you still run `install-deps` (the OS libraries aren't cached).
- **`--with-deps`** — installs the Linux system libraries browsers need; the usual cause of "works locally, fails in CI."
- **Secrets via `env`** — credentials/baseURL come from GitHub Secrets, never the repo ([auth](./09-authentication-and-storage-state.md)).
- **`fail-fast: false`** — one shard failing shouldn't cancel the others; you want the full picture.
- **`if: ${{ !cancelled() }}` on uploads** — publish artifacts even when tests failed (that's exactly when you need the report/traces).

## Artifacts: make failures debuggable

Always upload the **HTML report** and ensure traces are captured (`trace: 'on-first-retry'`). After a failed CI run you download the report artifact and open the trace ([trace viewer](./12-debugging-and-trace-viewer.md)) — that's how you debug a failure you can't reproduce locally. A suite whose CI failures aren't debuggable is barely better than no suite.

## CI as a release gate

This connects to the Git theme: protect `main` and make the Playwright job a **required status check** so a PR can't merge while tests fail. That turns the suite into an enforced quality gate rather than advisory. See the Git workflows topic on branch protection + CI gating.

## Interview Q&A

**Q: Walk through a robust Playwright CI pipeline.**
A: `actions/checkout` → `setup-node` with npm cache → `npm ci` → restore a browser-binary cache keyed on the Playwright version (run `install --with-deps` on a miss, `install-deps` on a hit) → run tests, usually sharded across a job matrix with `--shard=k/n` and `reporter: blob`, passing secrets via `env` → a dependent `merge-report` job downloads all blob artifacts and runs `merge-reports --reporter=html` → upload the HTML report (and traces) as artifacts, even on failure. Then make the job a required check on `main`.

**Q: Why cache browsers, and what's the cache key gotcha?**
A: Browser binaries are big and re-downloading them every run wastes minutes. The key must include the **Playwright version**, because the binaries are pinned to it — a stale cache after an upgrade gives version-mismatched browsers. Also, the cache holds binaries but not the OS-level libraries, so on a cache hit you still run `install-deps`.

**Q: Which reporters do you run in CI and why `blob`?**
A: I run multiple: `blob` for the mergeable per-shard report, `github` for inline PR annotations, often `junit` for the CI's native test UI. `blob` is essential under sharding — each shard emits one and a final job merges them into a single HTML report. Locally I use `html` + `list`. Reporters aren't mutually exclusive; you stack the ones each context needs.

**Q: How do you debug a CI-only failure?**
A: I rely on artifacts. With `trace: 'on-first-retry'` and the HTML report uploaded (even on failure, via `if: !cancelled()`), I download the report, open the failing test's trace, and inspect the before-snapshot/network/console at the failing action. The pipeline is designed so every failure leaves behind everything needed to diagnose it.

**Q: How does the suite become a real quality gate?**
A: Branch protection on `main` with the Playwright workflow as a *required* status check, so merges are blocked while tests fail — combined with PR-triggered runs. Optionally a smoke-tagged subset on every PR and the full sharded suite on merge to `main`, balancing feedback speed against coverage.

## Gotchas & Anti-patterns

- **Skipping `--with-deps` in CI** — missing system libraries; "browser failed to launch." Install OS deps.
- **Cache key without the Playwright version** — stale browsers after an upgrade. Always include the version in the key.
- **Not uploading artifacts on failure** — you lose the report/traces exactly when you need them. Use `if: ${{ !cancelled() }}`.
- **Forgetting the merge step under sharding** — N partial reports instead of one. Add a dependent `merge-reports` job.
- **Secrets in the repo/config** — leak permanently. Use CI secrets via `env`.
- **`fail-fast: true` on the shard matrix** — one shard's failure cancels the rest, hiding other failures. Set `fail-fast: false`.
- **A green suite that isn't a required check** — advisory tests get ignored. Gate merges on it via branch protection.

## Further Reading

- [Reporters](https://playwright.dev/docs/test-reporters)
- [Continuous Integration](https://playwright.dev/docs/ci)
- [GitHub Actions guide](https://playwright.dev/docs/ci-intro)
- [Sharding & merging reports](https://playwright.dev/docs/test-sharding)
