# Playwright

End-to-end testing with **Playwright + TypeScript** — the workbook's primary stack and the deepest theme. Coverage is tuned to what a Senior Test Automation Engineer is expected to reason about fluently: the web-first auto-waiting model and *why* it kills most flakiness, locator strategy, fixtures as the composition primitive, test architecture, network control, parallelism/sharding at scale, and turning a suite into a reliable CI gate. Every topic leads with the "why" and the trade-offs, not just the API surface.

## Topics (recommended reading order)

1. [Setup & Configuration](./01-setup-and-configuration.md) — `npm init playwright@latest`, project layout, and a deep dive on `playwright.config.ts`: `projects`, `use`, `baseURL`, `webServer`, timeouts, retries, trace/screenshot/video.
2. [Test Structure & the Runner](./02-test-structure-and-runner.md) — `test`/`test.describe`, hooks and their scope, annotations (`skip`/`fixme`/`fail`/`only`/`slow`), `test.step`, tags + `--grep`, test isolation by default.
3. [Locators](./03-locators.md) — locators vs ElementHandles, the `getByRole`-first priority list, chaining + `filter()`, strictness, and a resilient `data-testid` strategy.
4. [Actions & Auto-waiting](./04-actions-and-auto-waiting.md) — actionability checks, the web-first model that removes manual waits, `force`, and why fixed sleeps are the cardinal anti-pattern.
5. [Web-first Assertions](./05-web-first-assertions.md) — auto-retrying `expect(locator)` vs non-retrying `expect(value)`, key matchers, `expect.soft`, `expect.poll`, `expect.toPass`, timeouts.
6. [Fixtures](./06-fixtures.md) — built-in fixtures, custom fixtures via `test.extend`, test vs worker scope, `auto` fixtures, overrides and option fixtures; fixtures vs hooks.
7. [Page Object Model & Test Architecture](./07-page-object-model-and-architecture.md) — POMs built on locators, component objects, injecting POMs through fixtures, and avoiding over-abstraction.
8. [Network Mocking & Interception](./08-network-mocking-and-interception.md) — `route`/`fulfill`/`abort`/`continue`, modifying traffic, HAR record-and-replay, `waitForResponse`; mock vs real backend.
9. [Authentication & Storage State](./09-authentication-and-storage-state.md) — authenticate once with `storageState`, the `setup` project + dependencies, per-role state, and secrets handling.
10. [API Testing](./10-api-testing.md) — the `request` fixture and `APIRequestContext`, standalone API tests, hybrid UI+API for fast setup/teardown, and reusing auth.
11. [Parallelism, Retries & Sharding](./11-parallelism-retries-and-sharding.md) — workers, `fullyParallel`, isolation, `describe.serial`/`.parallel`, retries as a flaky signal, `--shard` and merging blob reports.
12. [Debugging & Trace Viewer](./12-debugging-and-trace-viewer.md) — UI mode, the Inspector/`--debug`, codegen, and reading a **trace** to diagnose a failure or flake post-mortem.
13. [Reporters & CI Integration](./13-reporters-and-ci-integration.md) — built-in reporters, an annotated GitHub Actions workflow, caching browsers, sharding in CI, and publishing artifacts.
14. [Flakiness & Stability](./14-flakiness-and-stability.md) — a cross-cutting playbook: the taxonomy of flake sources, the concrete fix for each, and why retries are a signal, not a cure.
