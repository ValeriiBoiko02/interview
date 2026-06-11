# Parallelism, Retries & Sharding

> How Playwright runs tests fast at scale — and the isolation and determinism trade-offs that come with it. This is the "how do you keep a 5,000-test suite under 10 minutes" conversation.

## TL;DR

- Playwright runs tests in parallel across **worker processes** (default ≈ half the CPU cores).
- `fullyParallel: true` parallelizes tests *within* a file too, not just across files.
- Each test gets an isolated context, so parallelism is safe **as long as tests are independent**.
- `test.describe.serial` forces order (and stops on first failure); `test.describe.configure({ mode })` tunes it per group.
- `retries` re-runs failures — a **flaky-detection signal**, not a fix.
- `--shard=k/n` splits the suite across CI machines; merge the per-shard **blob reports** into one.

## Workers and parallelism

```ts
// playwright.config.ts
export default defineConfig({
  workers: process.env.CI ? 4 : undefined, // undefined ≈ half the local cores
  fullyParallel: true,                      // tests within a file run in parallel too
});
```

A **worker** is a separate Node process running tests in its own browser. With `fullyParallel: false` (default), files run in parallel but tests *within* a file run sequentially; with `true`, individual tests parallelize across workers as well — usually the biggest wall-clock win. Tune workers to your CI machine's cores; too many causes resource contention and *adds* flakiness.

```bash
npx playwright test --workers=1     # serialize everything (debugging a flake)
npx playwright test --workers=50%   # half the cores
```

## Why parallelism is safe — and when it isn't

It's safe because each test runs in a fresh `BrowserContext` ([test structure](./02-test-structure-and-runner.md)) — no shared cookies/storage. It breaks when tests share **external** state the framework can't isolate: the same DB row, a singleton account, a fixed username, an order counter. Symptoms are order-dependent failures that vanish at `--workers=1`.

Fixes, in preference order:
1. Make each test create its own data (unique emails, API-seeded records) — see [API testing](./10-api-testing.md).
2. Use worker-scoped fixtures so each worker gets its own isolated resource ([fixtures](./06-fixtures.md)).
3. Only as a last resort, serialize the genuinely-coupled tests.

## Controlling execution mode per group

```ts
// Force order; ALL subsequent tests skip after the first failure (state builds up)
test.describe.serial('checkout wizard', () => {
  test('step 1: cart', async ({ page }) => { /* ... */ });
  test('step 2: shipping', async ({ page }) => { /* ... */ });
  test('step 3: payment', async ({ page }) => { /* ... */ });
});

// Explicitly parallelize a group even if fullyParallel is off
test.describe.configure({ mode: 'parallel' });

// Cap retries for one flaky-prone group
test.describe.configure({ retries: 2 });
```

`serial` is a smell to use sparingly: it sacrifices isolation and one failure cascades to skip the rest. Prefer independent tests. Reserve `serial` for a flow that genuinely cannot be decomposed (and even then, consider one test with `test.step`s instead).

## Retries — a signal, not a cure

```ts
export default defineConfig({ retries: process.env.CI ? 2 : 0 });
```

A retried test that passes is reported as **flaky** — green, but flagged. That flag is the point: it tells you which tests to investigate. Keep `retries: 0` locally so flakes surface immediately; allow a small number in CI so an infra blip doesn't fail the build, but treat a rising flaky count as a defect backlog, not background noise. Retries that routinely "rescue" a test are masking a real bug ([flakiness](./14-flakiness-and-stability.md)).

Use `trace: 'on-first-retry'` so the *first failing attempt* is captured for diagnosis without tracing every green run.

## Sharding across CI machines

```bash
# Machine 1 of 4, machine 2 of 4, ... — each runs a disjoint quarter
npx playwright test --shard=1/4
npx playwright test --shard=2/4
```

Sharding splits the test set across N parallel jobs for near-linear speedup. Each shard produces a **blob report**; you merge them into a single HTML report afterward:

```bash
# in each shard job:
npx playwright test --shard=${SHARD}/4 --reporter=blob
# in a final job, after collecting all blob-report/ artifacts:
npx playwright merge-reports --reporter=html ./all-blob-reports
```

`workers` parallelize *within* one machine; `--shard` parallelizes *across* machines. They compose: 4 shards × 4 workers ≈ 16-way parallelism. See [CI integration](./13-reporters-and-ci-integration.md) for the GitHub Actions matrix wiring.

## Interview Q&A

**Q: How does Playwright parallelize, and what makes it safe?**
A: It spreads tests across worker processes, each with its own browser; `fullyParallel` extends that to tests within a file. It's safe because every test runs in an isolated browser context — no shared cookies/storage by default — so tests are independent and order-free. The only thing the framework can't isolate is *external* shared state (DB, accounts), which is where you have to design for independence.

**Q: A test passes alone but fails under parallelism — how do you debug it?**
A: Classic shared-state interference. I confirm with `--workers=1` (if it passes serialized, it's a parallelism/isolation issue). Then I find the shared resource — a fixed user, a DB row, a counter — and make the test own its data (unique values, API-seeded records) or move the resource into a worker-scoped fixture. Serializing the tests is the last resort, not the first.

**Q: What's the difference between workers and shards?**
A: Workers are parallel processes *on one machine*; shards are disjoint slices of the suite run *across multiple machines/CI jobs*. They multiply: 4 shards each with 4 workers is ~16-way parallelism. Sharding is how you scale beyond a single machine's cores; you merge the per-shard blob reports into one report at the end.

**Q: What do retries actually buy you, and what's the danger?**
A: They re-run failed tests and mark a test that passes on retry as *flaky* — a green build plus a signal of which tests are unstable. The danger is treating retries as a fix: if you let flaky tests ride on retries indefinitely, you're shipping with hidden nondeterminism and eroding trust in the suite. Retries absorb rare infra blips; a persistent flake is a bug to fix.

**Q: When is `describe.serial` justified?**
A: Almost never for normal tests — it kills isolation and cascades failures (later tests skip after the first failure). It's only for a flow that genuinely can't be decomposed and must share state step-to-step, and even then a single test with `test.step` is often cleaner. If I'm using `serial` to work around shared data, I fix the data instead.

## Gotchas & Anti-patterns

- **Tests sharing a fixed account/DB row** — pass alone, fail in parallel. Generate unique data per test or isolate via worker fixtures.
- **Too many workers** — CPU/memory contention slows tests and *causes* flakiness. Match workers to available cores.
- **Relying on retries to keep a flaky test green** — masks a real bug and rots trust. Investigate flagged-flaky tests.
- **`describe.serial` as a default** — sacrifices isolation; one failure skips the rest. Prefer independent tests.
- **Forgetting to merge blob reports** — you get N partial reports instead of one. Use `merge-reports` in a final job.
- **`retries > 0` locally** — hides flakes during development. Keep local retries at 0; allow a few only in CI.
- **Assuming `beforeAll` runs once globally under sharding/parallelism** — it's per worker (and per shard machine). Use setup projects for once-only work.

## Further Reading

- [Parallelism](https://playwright.dev/docs/test-parallel)
- [Sharding](https://playwright.dev/docs/test-sharding)
- [Retries](https://playwright.dev/docs/test-retries)
- [`merge-reports`](https://playwright.dev/docs/test-sharding#merging-reports-from-multiple-shards)
