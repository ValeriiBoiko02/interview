# Test Structure & the Runner

> How Playwright's runner organizes tests: declaring tests, grouping, hooks and their scope, annotations, steps, and tagging — plus the isolation model that underpins all of it.

## TL;DR

- `test('name', async ({ page }) => {...})` declares a test; `test.describe` groups them.
- **Each test gets a fresh `BrowserContext`** — isolation is the default, not something you arrange.
- Hooks: `beforeEach`/`afterEach` run per test; `beforeAll`/`afterAll` run **once per worker**, not once globally.
- Annotations: `test.skip`, `test.fixme`, `test.fail`, `test.slow`, `test.only` — some accept a condition.
- `test.step()` groups actions for readable reports/traces. Tags (`{ tag: '@smoke' }`) + `--grep` filter what runs.
- Prefer fixtures over heavy hooks for setup that returns a value (see [fixtures](./06-fixtures.md)).

## Declaring and grouping tests

```ts
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');           // baseURL-relative; runs before each test below
  });

  test('rejects an invalid password', async ({ page }) => {
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('wrong');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page.getByRole('alert')).toHaveText('Invalid credentials');
  });

  test('signs in with valid credentials', async ({ page }) => {
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('correct-horse');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page).toHaveURL('/dashboard');
  });
});
```

`test.describe` is for grouping and shared hooks/config — it does **not** create a shared browser state between its tests. Each test still gets its own context.

## Test isolation — the model that prevents most cross-test flakiness

Every test runs in a brand-new `BrowserContext` (think a fresh incognito profile): no shared cookies, localStorage, or session. The `page` fixture is a new page in that context. This is why tests are safe to parallelize and reorder by default.

Consequences worth saying out loud:
- You **cannot** rely on test A leaving state for test B. If you need shared expensive setup (e.g. a logged-in session), use `storageState` ([auth](./09-authentication-and-storage-state.md)) or a worker-scoped fixture ([fixtures](./06-fixtures.md)) — not test ordering.
- Because isolation is per-test, `fullyParallel` and sharding ([parallelism](./11-parallelism-retries-and-sharding.md)) are safe by default.

## Hooks and their scope — the common trap

```ts
test.describe('Reports', () => {
  test.beforeAll(async ({ browser }) => {
    // runs ONCE PER WORKER, not once for the whole run.
    // With 4 workers, this body executes up to 4 times.
  });

  test.beforeEach(async ({ page }) => {
    // runs before every test in this describe — the right place for per-test navigation/setup
  });

  test.afterEach(async ({ page }, testInfo) => {
    if (testInfo.status !== testInfo.expectedStatus) {
      // e.g. attach extra diagnostics on failure
    }
  });
});
```

`beforeAll`/`afterAll` are **per worker**, not global. If you assume "once for the entire suite" you'll get surprising behavior under parallelism. For truly-once setup across the whole run, use a `setup` **project dependency** or `globalSetup` in the config. Prefer fixtures for setup that produces a value a test consumes — they compose and tear down automatically, unlike hooks which only mutate shared scope.

## Annotations

```ts
test.skip('not implemented yet', async ({ page }) => { /* never runs */ });

// conditional skip — evaluated at runtime
test('webkit-only behavior', async ({ page, browserName }) => {
  test.skip(browserName !== 'webkit', 'Safari-specific rendering');
  // ...
});

test.fixme('flaky: needs rework', async ({ page }) => { /* skipped, flagged as a known bug */ });

test.fail('documents a known server bug', async ({ page }) => {
  // this test is EXPECTED to fail; the runner errors if it unexpectedly passes
});

test('huge data grid', async ({ page }) => {
  test.slow(); // triples the timeout for this legitimately slow test
});

test.only('focus while developing', async ({ page }) => { /* set forbidOnly in CI! */ });
```

| Annotation | Meaning | Note |
|---|---|---|
| `skip` | Don't run. | Conditional form takes `(condition, reason)`. |
| `fixme` | Don't run; marks a known-broken test. | Signals "needs fixing", semantically richer than `skip`. |
| `fail` | Expected to fail; **passing is an error**. | Documents a real bug and alerts you when it's fixed. |
| `slow` | Triple the timeout. | For genuinely slow work, not to paper over bad waits. |
| `only` | Run just this. | Guard with `forbidOnly` in CI ([setup](./01-setup-and-configuration.md)). |

## Steps and tags

```ts
test('checkout flow', { tag: ['@smoke', '@checkout'] }, async ({ page }) => {
  await test.step('add item to cart', async () => {
    await page.getByRole('button', { name: 'Add to cart' }).click();
    await expect(page.getByTestId('cart-count')).toHaveText('1');
  });

  await test.step('complete payment', async () => {
    await page.getByRole('button', { name: 'Checkout' }).click();
    await expect(page).toHaveURL('/order-confirmation');
  });
});
```

```bash
npx playwright test --grep @smoke          # run only smoke-tagged tests
npx playwright test --grep-invert @slow    # exclude slow ones
```

`test.step` makes the HTML report and trace readable — failures point to the failing step, and the timeline groups actions. Tags + `--grep` give you suite slices (smoke vs full) without restructuring files.

## Interview Q&A

**Q: How does Playwright isolate tests, and why does it matter?**
A: Each test runs in its own `BrowserContext` — a clean slate with no shared cookies, storage, or cache, plus a fresh `page`. That makes tests independent and order-free, which is exactly what lets the runner parallelize within a file (`fullyParallel`) and shard across machines safely. The flip side: you must never depend on another test's leftover state; share expensive setup via `storageState` or fixtures instead.

**Q: `beforeAll` — once per suite or once per worker?**
A: Once per *worker*. With N workers it can run up to N times. People who assume it's global get bitten when parallelism is on. For genuinely once-per-run work, use a `setup` project dependency or `globalSetup`.

**Q: When do you reach for a fixture instead of `beforeEach`?**
A: When setup *produces a value* the test uses (a logged-in page, a seeded user, a POM) or needs guaranteed teardown. Fixtures are lazy (only run if requested), composable, and tear down in reverse order automatically. Hooks just run imperatively against shared scope and can't return a typed value cleanly. See [fixtures](./06-fixtures.md).

**Q: Difference between `test.skip`, `test.fixme`, and `test.fail`?**
A: `skip` = don't run (optionally conditionally). `fixme` = don't run, and it's flagged as known-broken/needs-rework. `fail` = *do* run but it's expected to fail; the runner errors if it unexpectedly passes — useful to track a known product bug and get alerted when it's fixed.

## Gotchas & Anti-patterns

- **Assuming `beforeAll` is global** — it's per worker. Use a setup project / `globalSetup` for once-only work.
- **Sharing state between tests via order** — breaks under parallelism and reordering; isolation is intentional. Use fixtures/`storageState`.
- **Committing `test.only`** — runs just that test and "passes" CI while skipping everything. Set `forbidOnly` in CI.
- **Putting per-test navigation in `beforeAll`** — it only runs once per worker, so most tests start on the wrong page. Per-test setup goes in `beforeEach` or a fixture.
- **Overusing `test.slow()`/timeouts to fix flakes** — usually masks a bad wait; web-first assertions ([assertions](./05-web-first-assertions.md)) are the real fix.

## Further Reading

- [Writing tests](https://playwright.dev/docs/writing-tests)
- [Annotations](https://playwright.dev/docs/test-annotations)
- [Test steps & tags](https://playwright.dev/docs/test-annotations#tag-tests)
- [Parameterize / projects](https://playwright.dev/docs/test-parameterize)
