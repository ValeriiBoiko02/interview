# Fixtures

> Fixtures are Playwright's dependency-injection system and the composition primitive a senior suite is built on. They replace ad-hoc `beforeEach` setup with lazy, typed, auto-cleaned building blocks.

## TL;DR

- A fixture is a named, on-demand resource a test requests by destructuring it: `async ({ page }) => {}`.
- **Built-in fixtures:** `page`, `context`, `browser`, `request`, `browserName`, plus options like `baseURL`.
- **Custom fixtures** via `test.extend` — set up before, `use(value)`, tear down after. Composable and typed.
- **Scope:** `test` (default, per test) vs `worker` (shared across tests in a worker, for expensive setup).
- Fixtures are **lazy** (only instantiated if a test requests them) and tear down in reverse order automatically.
- Prefer fixtures over hooks when setup *produces a value* or needs guaranteed cleanup.

## Built-in fixtures

```ts
test('uses built-ins', async ({ page, context, browser, request, browserName }) => {
  // page    — a fresh Page in an isolated context (the one you use 95% of the time)
  // context — the BrowserContext (cookies/storage); make extra pages/tabs here
  // browser — the shared Browser instance (worker-scoped)
  // request — an APIRequestContext for API calls (see API testing topic)
  // browserName — 'chromium' | 'firefox' | 'webkit'
});
```

You rarely create a browser/context yourself — requesting `page` gives you an isolated one for free. Request `context` when you need a second tab or to manipulate cookies/permissions; request `browser` only for advanced multi-context scenarios.

## Custom fixtures with `test.extend`

```ts
// fixtures.ts — define once, import everywhere instead of @playwright/test
import { test as base, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page';

type MyFixtures = {
  loginPage: LoginPage;
  authedPage: import('@playwright/test').Page;
};

export const test = base.extend<MyFixtures>({
  // a POM fixture — built per test, no manual instantiation in the test body
  loginPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await use(loginPage);            // hand the value to the test
    // (teardown after use(), if any, goes here)
  },

  // a fixture that depends on another fixture (composition)
  authedPage: async ({ page, baseURL }, use) => {
    await page.goto(`${baseURL}/login`);
    await page.getByLabel('Email').fill(process.env.TEST_USER!);
    await page.getByLabel('Password').fill(process.env.TEST_PASS!);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page).toHaveURL('/dashboard');
    await use(page);
  },
});

export { expect };
```

```ts
// a spec — requests exactly the fixtures it needs
import { test, expect } from '../fixtures';

test('shows the user menu when logged in', async ({ authedPage }) => {
  await expect(authedPage.getByRole('button', { name: 'Account' })).toBeVisible();
});
```

The shape: `name: async ({ deps }, use) => { /* setup */ await use(value); /* teardown */ }`. Everything before `use` is setup, the value passed to `use` is what the test receives, and code after `use` runs as teardown. Fixtures can depend on other fixtures, so they compose into a graph the runner resolves automatically.

## Scope: `test` vs `worker`

```ts
type WorkerFixtures = { seededDb: DbHandle };

export const test = base.extend<{}, WorkerFixtures>({
  seededDb: [async ({}, use) => {
    const db = await seedExpensiveDataset();   // runs ONCE per worker, not per test
    await use(db);
    await db.cleanup();
  }, { scope: 'worker' }],
});
```

| Scope | Lifecycle | Use for |
|---|---|---|
| `test` (default) | New instance per test. | Anything test-specific: POMs, per-test data, the `page`. |
| `worker` | One instance shared by all tests in that worker process. | Expensive, reusable setup: a DB seed, a server handle, a shared auth token. |

Worker scope trades isolation for speed. Only use it for things genuinely safe to share — if tests can mutate it and interfere, you've reintroduced cross-test coupling. Note: with N workers a worker fixture still runs N times, not once.

## `auto` fixtures and option fixtures

```ts
export const test = base.extend<{ timezone: string }>({
  // option fixture: a configurable default, overridable via test.use / project use
  timezone: ['UTC', { option: true }],

  // auto fixture: runs for every test even if not explicitly requested
  attachConsoleLogs: [async ({ page }, use, testInfo) => {
    const logs: string[] = [];
    page.on('console', m => logs.push(m.text()));
    await use();
    if (testInfo.status !== testInfo.expectedStatus) {
      await testInfo.attach('console.log', { body: logs.join('\n') });
    }
  }, { auto: true }],
});
```

- **Option fixtures** (`{ option: true }`) define configurable values you can override in `test.use({...})` or a project's `use` — how you parameterize a suite (e.g. role, locale, feature flag).
- **Auto fixtures** (`{ auto: true }`) run for every test without being requested — good for cross-cutting concerns like attaching logs/diagnostics on failure.

## Overriding built-in fixtures

```ts
export const test = base.extend({
  // override the built-in `page` to start every test already on the homepage
  page: async ({ page }, use) => {
    await page.goto('/');
    await use(page);
  },
});
```

You can override `page`, `context`, etc., wrapping the built-in to inject default behavior suite-wide. Powerful, but use sparingly — overriding `page` affects *every* test and can surprise readers.

## Fixtures vs hooks

| | Fixture | `beforeEach`/`beforeAll` |
|---|---|---|
| Returns a value | Yes (typed, injected) | No (mutates shared scope) |
| Lazy | Yes — only runs if requested | No — always runs |
| Composition | Depends on other fixtures | Manual ordering |
| Teardown | Automatic, reverse order | Separate `afterEach` you must keep in sync |
| Best for | Resources tests consume (POMs, sessions, data) | Simple imperative side-effects (navigate, reset) |

Reach for a fixture when setup produces something the test uses or needs reliable cleanup; a `beforeEach` is fine for a one-line navigate.

## Interview Q&A

**Q: What problem do fixtures solve over `beforeEach`?**
A: They give you lazy, typed, composable setup with automatic teardown. A fixture *returns a value* the test consumes (a POM, an authenticated page, seeded data), only runs if a test actually requests it, can depend on other fixtures, and tears down in reverse order without a matching `afterEach`. Hooks just run imperatively against shared scope and can't cleanly hand back a typed value. Fixtures scale; sprawling hooks don't.

**Q: Explain test vs worker scope and the trade-off.**
A: Test-scoped fixtures are recreated per test — maximum isolation, the default. Worker-scoped fixtures are created once per worker process and shared by every test that worker runs — for expensive setup like a DB seed or auth token. The trade-off is isolation vs speed: worker scope is only safe for things tests won't mutate into each other's way. And "once per worker" means N times with N workers, not globally once.

**Q: How do `auto` and `option` fixtures differ?**
A: An `auto` fixture runs for every test whether or not it's requested — ideal for cross-cutting behavior like attaching a trace/logs on failure. An `option` fixture defines a configurable default value (e.g. role/locale) that you override via `test.use()` or a project's `use`, which is how you parameterize the same tests across configurations.

**Q: Why export your own `test` from a fixtures file?**
A: `test.extend` returns a new `test` carrying your custom fixtures and types. Specs import *that* instead of `@playwright/test`, so every test has access to your POMs/sessions with full type-safety, and there's one place to evolve setup. It's the backbone of a maintainable suite.

**Q: How does teardown work?**
A: Anything after `await use(value)` in a fixture is teardown, and the runner runs teardowns in reverse dependency order automatically — even if the test failed. That's a key advantage over hooks, where you maintain a separate `afterEach` and risk it drifting out of sync.

## Gotchas & Anti-patterns

- **Putting expensive setup in `beforeEach`** — re-runs every test. If it's reusable and safe to share, make it a worker-scoped fixture.
- **Worker-scoped fixtures holding mutable state tests change** — reintroduces cross-test coupling and order-dependence. Keep worker fixtures read-only/idempotent.
- **Instantiating POMs in every test body (`new LoginPage(page)`)** — boilerplate that drifts. Inject them as fixtures.
- **Importing `test` from `@playwright/test` in specs that need custom fixtures** — they won't exist. Import your extended `test`.
- **Overusing auto fixtures** — they run everywhere and can slow the whole suite or hide behavior. Reserve for genuine cross-cutting concerns.
- **Forgetting `await use(...)`** — the test gets `undefined` / setup-teardown won't bracket correctly. The `use` call is mandatory and marks the setup/teardown boundary.

## Further Reading

- [Fixtures](https://playwright.dev/docs/test-fixtures)
- [Worker-scoped fixtures](https://playwright.dev/docs/test-fixtures#worker-scoped-fixtures)
- [Parameterize with option fixtures](https://playwright.dev/docs/test-parameterize)
