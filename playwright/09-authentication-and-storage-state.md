# Authentication & Storage State

> Logging in through the UI in every test is slow and flaky. The senior pattern is to authenticate **once**, save the session, and reuse it — with clean handling for multiple roles and secrets.

## TL;DR

- `storageState` captures cookies + localStorage to a JSON file; reuse it so tests start already logged in.
- The standard wiring: a **`setup` project** that logs in and writes the state, and real projects that depend on it (`dependencies: ['setup']`) and `use` the saved state.
- **One state per role** (admin, user, guest); select via project or `test.use({ storageState })`.
- Keep credentials in **env vars/secrets**, never in the repo. The state file itself contains a live session — gitignore it.
- For most tests, **don't log in through the UI** — restore state. Keep a couple of real login tests to cover the login flow itself.

## The setup-project pattern (recommended)

```ts
// playwright.config.ts (excerpt)
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /global\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],   // 'setup' runs first; its saved state is reused here
    },
  ],
});
```

```ts
// tests/global.setup.ts — runs once before the dependent projects
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER!);
  await page.getByLabel('Password').fill(process.env.TEST_PASS!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL('/dashboard');           // confirm login succeeded

  await page.context().storageState({ path: authFile }); // persist cookies + localStorage
});
```

Now every test in `chromium` starts authenticated — no per-test login. The `setup` project is just a test file that happens to write a state file; `dependencies` guarantees ordering and that the state exists before dependents run. This is preferred over the older `globalSetup` function because it shows in the report, gets traces, and uses the same fixtures as normal tests.

## Multiple roles

```ts
// config: a project (and auth file) per role
projects: [
  { name: 'setup', testMatch: /auth\.setup\.ts/ },
  { name: 'admin', use: { storageState: 'playwright/.auth/admin.json' }, dependencies: ['setup'] },
  { name: 'user',  use: { storageState: 'playwright/.auth/user.json'  }, dependencies: ['setup'] },
],
```

```ts
// auth.setup.ts — write one state file per role
setup('auth as admin', async ({ page }) => { /* login as admin */ await page.context().storageState({ path: 'playwright/.auth/admin.json' }); });
setup('auth as user',  async ({ page }) => { /* login as user  */ await page.context().storageState({ path: 'playwright/.auth/user.json'  }); });
```

To use a specific role inside one spec regardless of project:

```ts
test.use({ storageState: 'playwright/.auth/admin.json' });

test('admin can delete a user', async ({ page }) => {
  await page.goto('/admin/users');
  await expect(page.getByRole('button', { name: 'Delete' })).toBeVisible();
});
```

For a test that needs **two roles simultaneously** (e.g. admin acts, user observes), create extra contexts:

```ts
test('admin change is visible to the user', async ({ browser }) => {
  const adminCtx = await browser.newContext({ storageState: 'playwright/.auth/admin.json' });
  const userCtx  = await browser.newContext({ storageState: 'playwright/.auth/user.json' });
  const admin = await adminCtx.newPage();
  const user  = await userCtx.newPage();
  // ...drive both pages, assert cross-effects...
  await adminCtx.close(); await userCtx.close();
});
```

## Faster auth: skip the UI entirely

If your app issues a token via an API, log in over HTTP instead of clicking through the form — even faster and more robust:

```ts
setup('authenticate via API', async ({ request }) => {
  const res = await request.post('/api/login', {
    data: { email: process.env.TEST_USER, password: process.env.TEST_PASS },
  });
  expect(res.ok()).toBeTruthy();
  // persist whatever the app uses (cookie set by the response, or a token in localStorage)
  await request.storageState({ path: 'playwright/.auth/user.json' });
});
```

Keep **one** UI-driven login test to actually exercise the login form; everyone else restores state. See [API testing](./10-api-testing.md) for the `request` context.

## Secrets & the state file

- Credentials come from **env vars** (`process.env.TEST_USER`) populated from CI secrets or a gitignored `.env` (loaded via `dotenv`). Never commit them.
- The generated `playwright/.auth/*.json` contains a **live session** (cookies/tokens). Add `playwright/.auth/` to `.gitignore` — committing it leaks a valid session and goes stale anyway.
- In CI, regenerate the state each run (the `setup` project does this). Sessions expire; don't cache auth state across days.

## Interview Q&A

**Q: How do you avoid logging in through the UI in every test?**
A: Authenticate once in a `setup` project, save the session with `storageState` to a JSON file (cookies + localStorage), and have the real projects `dependencies: ['setup']` and `use: { storageState }`. Every test then starts already logged in. It cuts thousands of redundant logins and removes the login form as a flakiness source for unrelated tests.

**Q: `setup` project vs `globalSetup` — why prefer the project?**
A: A setup project is a normal test: it appears in the report, produces a trace if it fails, uses the same fixtures and config, and integrates with `dependencies` for ordering and per-role parallelism. `globalSetup` is an opaque function with no trace and a different execution model. The project approach is the current recommended pattern.

**Q: How do you handle multiple roles?**
A: One state file per role, written by separate setup tests, and a project per role (or `test.use({ storageState })` per spec). For a single test exercising two roles at once, I create two browser contexts with different `storageState` and drive a page in each — contexts are cheap and fully isolated.

**Q: How do you keep credentials and sessions out of the repo?**
A: Credentials live in env vars sourced from CI secrets (or a gitignored `.env`), referenced as `process.env.*`. The generated auth state file holds a live session, so `playwright/.auth/` is gitignored and regenerated each CI run rather than cached — sessions expire and committing one is a security leak.

**Q: Should you still test the login flow if you bypass it everywhere?**
A: Yes — keep a small number of tests that log in through the actual UI (and ideally a few negative cases) so the login form itself is covered. The `storageState` optimization is for the *other* tests that merely need to *be* authenticated, not to *test* authentication.

## Gotchas & Anti-patterns

- **Logging in through the UI in every test** — slow and turns the login form into a shared flakiness source. Restore `storageState` instead.
- **Committing `playwright/.auth/*.json`** — leaks a live session and goes stale. Gitignore it; regenerate per run.
- **Hardcoding credentials in specs/config** — secrets in history forever. Use env vars / CI secrets.
- **Caching auth state across runs/days** — sessions expire, causing confusing mass failures. Regenerate in the `setup` project each run.
- **Reusing one logged-in context across tests to "save time"** — breaks isolation; one test's actions bleed into another. Each test gets its own context seeded from the *file*.
- **Forgetting `dependencies: ['setup']`** — tests run before the state file exists and fail unauthenticated.

## Further Reading

- [Authentication](https://playwright.dev/docs/auth)
- [`storageState`](https://playwright.dev/docs/api/class-browsercontext#browser-context-storage-state)
- [Project dependencies](https://playwright.dev/docs/test-projects#dependencies)
