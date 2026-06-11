# Setup & Configuration

> **Goal:** scaffold a Playwright + TypeScript project from scratch and understand `playwright.config.ts` well enough to defend every option in an interview.

## Quick reference

```bash
# scaffold a new project (interactive: TS, test folder, GH Actions, browsers)
npm init playwright@latest

# install / update browsers (and OS deps in CI)
npx playwright install                 # browsers only
npx playwright install --with-deps     # browsers + Linux system libs (CI)
npx playwright install chromium        # just one

# run
npx playwright test                    # all tests, headless, all projects
npx playwright test --ui               # UI mode (best local DX)
npx playwright test --headed --project=chromium
npx playwright test login.spec.ts -g "invalid password"
npx playwright test --debug            # step through with the Inspector
npx playwright show-report             # open the last HTML report
```

`npm init playwright@latest` creates `playwright.config.ts`, a `tests/` folder with an example spec, a `.github/workflows/playwright.yml`, and installs `@playwright/test`. Playwright ships its **own** test runner — you do not pair it with Jest/Mocha.

## How it fits together

- **`@playwright/test`** is both the runner and the assertion library. Import `test` and `expect` from it — not from `playwright` (that's the lower-level library without the runner/fixtures).
- **Browsers are downloaded binaries**, not npm packages. They live outside `node_modules` (in a cache dir), so CI must run `playwright install` separately and should cache that dir.
- **Config is the contract** for how, where, and against what your tests run. Most interview "how would you scale this" answers route back to a config option.

## `playwright.config.ts` — the deep dive

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',              // where specs live
  fullyParallel: true,             // run tests within a file in parallel too
  forbidOnly: !!process.env.CI,    // fail the build if a stray test.only is committed
  retries: process.env.CI ? 2 : 0, // retry only in CI; surfaces flaky vs broken
  workers: process.env.CI ? 4 : undefined, // undefined = ~half the CPU cores locally

  timeout: 30_000,                 // per-test timeout (ms)
  expect: { timeout: 5_000 },      // per-assertion auto-retry budget

  reporter: process.env.CI
    ? [['blob'], ['github']]       // blob = mergeable across shards; github = PR annotations
    : [['html', { open: 'never' }], ['list']],

  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',       // capture a trace only when a test is retried
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    actionTimeout: 0,              // 0 = no per-action cap; rely on test timeout
  },

  projects: [
    { name: 'setup', testMatch: /global\.setup\.ts/ },        // see Auth topic
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],     // run 'setup' first; reuse its storageState
    },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] }, dependencies: ['setup'] },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] },  dependencies: ['setup'] },
    { name: 'mobile',   use: { ...devices['Pixel 7'] },         dependencies: ['setup'] },
  ],

  // start the app under test before the suite, reuse it locally
  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

### The options that matter most, and why

| Option | What it does | Why a senior cares |
|---|---|---|
| `projects` | Named run configurations — browsers, devices, or logical groups — each with their own `use`. | The backbone of cross-browser, multi-role, and setup/teardown via `dependencies`. Sharding and `--project` filtering build on it. |
| `use` (top-level) | Default context/page options for every project (baseURL, trace, viewport…). | Project-level `use` overrides it; `test.use()` overrides per-file. Knowing the precedence is a common probe. |
| `baseURL` | Lets `page.goto('/login')` resolve against it. | Keeps specs environment-agnostic; switch envs via one env var, not edits. |
| `webServer` | Boots (and waits for) the app before tests. | Removes "did you start the server?" flakiness; `reuseExistingServer` keeps local runs fast. |
| `retries` | Re-runs failed tests N times. | Distinguishes *flaky* (passes on retry) from *broken*. A retry is a signal to investigate, not a fix — see [flakiness](./14-flakiness-and-stability.md). |
| `timeout` vs `expect.timeout` | Whole-test budget vs per-assertion auto-retry budget. | They're different clocks. A slow assertion can blow the test timeout without ever hitting the expect timeout. |
| `trace` | When to record a full trace (DOM, network, console, actions). | `'on-first-retry'` is the standard: near-zero overhead on green runs, a full post-mortem on the failures that matter. See [trace viewer](./12-debugging-and-trace-viewer.md). |
| `fullyParallel` | Parallelizes tests *within* a file, not just across files. | Big wall-clock win — but only safe if tests are independent. See [parallelism](./11-parallelism-retries-and-sharding.md). |
| `forbidOnly` | Fails the run if `test.only` is present. | Stops a committed `.only` from silently skipping the whole suite in CI. |

### Config resolution order (override precedence)

`test.use({...})` in a file/`describe` **>** project-level `use` **>** top-level `use` **>** Playwright defaults. Mention this when asked "how do you run one suite against a different baseURL/role" — the answer is `test.use`, not a second config.

## TypeScript notes

- `npm init` sets up TS with no extra build step — Playwright transpiles specs on the fly. You generally don't need a separate `tsc` build for tests.
- Type your fixtures and POMs; export an extended `test` from a fixtures file and import *that* everywhere (see [fixtures](./06-fixtures.md)). This is what keeps a large suite strictly typed.
- A `tsconfig.json` `paths` alias (e.g. `@pages/*`) keeps imports clean across a growing suite.

## Interview Q&A

**Q: Why does Playwright bundle its own test runner instead of using Jest?**
A: The runner is fixture-aware and parallelism-aware in a browser-specific way: it manages browser/context lifecycles, isolates each test in a fresh context, runs `projects` across browsers, handles retries/trace/sharding, and provides web-first `expect`. Bolting Playwright onto Jest means reimplementing all of that and losing fixtures, projects, and the trace integration.

**Q: What's the difference between the per-test `timeout` and `expect.timeout`?**
A: `timeout` is the budget for the *entire* test (default 30s); if exceeded the test fails regardless of where it is. `expect.timeout` (default 5s) is how long a single auto-retrying assertion polls before failing. Actions like `click` have their own `actionTimeout`. They're independent clocks — a test can fail on the overall timeout while no single assertion ever timed out.

**Q: How do you run the same suite against staging vs production?**
A: Parameterize `baseURL` (and any creds) via env vars read in the config, and keep specs path-relative (`page.goto('/checkout')`). One config, switched by `BASE_URL=... npx playwright test`. For per-run option changes within a suite, `test.use()`. Don't fork the config per environment.

**Q: Why install browsers separately, and how do you handle that in CI?**
A: Browsers are large binaries managed by Playwright outside npm, version-pinned to the `@playwright/test` release. In CI you run `npx playwright install --with-deps` (the `--with-deps` pulls Linux system libraries) and cache the browser directory keyed on the Playwright version to avoid re-downloading every run. See [CI integration](./13-reporters-and-ci-integration.md).

## Gotchas & Anti-patterns

- **Importing from `playwright` instead of `@playwright/test`** — you lose `test`, fixtures, and web-first `expect`. Always `import { test, expect } from '@playwright/test'`.
- **Forgetting `playwright install` in CI** — tests fail with "browser not found". It's a separate step from `npm ci`.
- **`retries` masking real bugs** — if a test only passes on retry, it's flaky; treat that as a defect to fix, not green. Keep `retries: 0` locally so flakes are visible.
- **Setting a huge global `timeout` to "fix" slowness** — hides the real problem (a bad wait or slow selector) and makes failing tests take forever. Fix the wait, don't pad the budget.
- **Committing `test.only`** — silently runs just that one test. Set `forbidOnly: !!process.env.CI` so CI rejects it.
- **`reuseExistingServer: true` in CI** — can attach to a stale server. Gate it to local only (`!process.env.CI`).

## Further Reading

- [Playwright — Installation](https://playwright.dev/docs/intro)
- [Test configuration](https://playwright.dev/docs/test-configuration)
- [`devices` registry](https://playwright.dev/docs/emulation)
- [`webServer`](https://playwright.dev/docs/test-webserver)
