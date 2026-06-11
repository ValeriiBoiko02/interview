# Debugging & Trace Viewer

> Playwright's debugging tooling is a genuine differentiator. The trace viewer in particular is how you diagnose a CI failure you can't reproduce locally — a frequent senior interview topic.

## TL;DR

- **UI mode** (`--ui`) is the best local loop: watch mode, time-travel, live locator picker.
- **`--debug`** opens the Inspector to step through with `PWDEBUG`; pause at a point with `await page.pause()`.
- **Codegen** (`npx playwright codegen`) records actions into a starting script and suggests locators.
- **Trace viewer** is the post-mortem: a full timeline with DOM snapshots, network, console, and the action log — ideal for CI failures.
- Configure `trace: 'on-first-retry'` so failures get a trace with near-zero cost on green runs.

## Local debugging tools

```bash
npx playwright test --ui                 # UI mode: pick tests, watch, time-travel, locator picker
npx playwright test login.spec.ts --debug  # step through with the Playwright Inspector
npx playwright codegen http://localhost:3000  # record clicks into a script + locator suggestions
npx playwright show-report               # open the last HTML report
npx playwright test --last-failed        # re-run only the tests that failed last time
npx playwright test -g "checkout" --headed --workers=1  # watch one test run, serialized
```

```ts
// Pause execution and open the Inspector at a precise point:
test('debug a step', async ({ page }) => {
  await page.goto('/checkout');
  await page.pause();                    // Inspector opens here; resume manually, try locators live
  await page.getByRole('button', { name: 'Pay' }).click();
});
```

**UI mode** is the day-to-day tool: it shows each action with a before/after DOM snapshot, lets you hover the timeline (time-travel), re-runs on save (watch mode), and has a **pick locator** button that generates a recommended locator from a click — directly reinforcing the [locator priority](./03-locators.md). `--debug`/`page.pause()` is for stepping line-by-line.

## The trace viewer — diagnosing failures you can't reproduce

A **trace** is a self-contained recording of a test run: every action, before/after DOM snapshots, network requests, console logs, source, and timings. It's the answer to "it fails in CI but passes on my machine."

```ts
// playwright.config.ts
use: {
  trace: 'on-first-retry',   // record only when a test is retried (i.e. when it first failed)
}
```

```bash
# open a trace produced by CI (download the artifact first)
npx playwright show-trace trace.zip
```

| `trace` setting | When it records | Use |
|---|---|---|
| `'off'` | never | when you don't need traces |
| `'on'` | every test | heavy; local deep-dives only |
| `'retain-on-failure'` | keeps trace for failed tests | good CI default if not using retries |
| `'on-first-retry'` | records the first retry attempt | **recommended**: failures get a trace, green runs cost nothing |

### How to read a trace

1. **Timeline** at the top — scrub through; the failing action is highlighted in red.
2. **Action list** (left) — click any action to see the DOM **snapshot** exactly as it was *before* and *after*. This reveals whether the element was present/visible/covered at that moment.
3. **Network / Console / Log tabs** — correlate a failure with a 500 response, a JS error, or a slow request.
4. **Source + Call** — the exact line and the locator that was used.

The workflow: open the trace, jump to the red action, inspect the before-snapshot. You'll almost always see *why* the actionability check failed — an overlay covering the button, a spinner still showing, the element not yet rendered ([actions & auto-waiting](./04-actions-and-auto-waiting.md)).

## Other diagnostics

```ts
// Attach custom artifacts on failure (logs, screenshots, API dumps)
test('with diagnostics', async ({ page }, testInfo) => {
  // ... test body ...
  if (testInfo.status !== testInfo.expectedStatus) {
    await testInfo.attach('page-html', { body: await page.content(), contentType: 'text/html' });
  }
});
```

Config-level `screenshot: 'only-on-failure'` and `video: 'retain-on-failure'` ([setup](./01-setup-and-configuration.md)) give lightweight failure evidence in the HTML report, but the **trace** is strictly more powerful — prefer it for real diagnosis. The **VS Code extension** adds run/debug from the gutter, a live locator picker, and trace viewing inside the editor.

## Interview Q&A

**Q: A test fails in CI but you can't reproduce it locally. How do you diagnose it?**
A: Open its trace. With `trace: 'on-first-retry'`, the failed run is recorded as a zip artifact; `npx playwright show-trace trace.zip` gives me the timeline, per-action DOM snapshots, network, and console. I jump to the red (failing) action and inspect the before-snapshot — that usually shows the real cause: an overlay intercepting the click, a spinner still up, a 500 in the network tab, or the element not yet rendered. It turns "flaky in CI" into a concrete root cause without reproducing it.

**Q: What's in a trace, and what's the recommended capture setting?**
A: A trace is a complete recording: action log, before/after DOM snapshots, network, console, source, and timings — self-contained in a zip. The recommended setting is `trace: 'on-first-retry'`: green runs record nothing (no overhead), but any test that fails and retries produces a full trace. `retain-on-failure` is the alternative when you don't use retries; `'on'` is too heavy for routine use.

**Q: UI mode vs trace viewer — when each?**
A: UI mode is for *developing and debugging locally* — watch mode, time-travel, the locator picker, instant feedback. The trace viewer is for *post-mortem* analysis of a run that already happened (especially in CI). They share the snapshot/time-travel UX, but one is interactive-live and the other is forensic.

**Q: How does codegen fit a senior workflow?**
A: As a starting point and a locator suggester, not a finished test. I use it to bootstrap a flow and to see what locators Playwright recommends (it follows the role/label/testid priority), then I refactor: extract POMs, add real assertions, remove brittle steps. Shipping raw codegen output is the anti-pattern.

**Q: Screenshots/videos vs traces?**
A: Screenshots and video give quick visual evidence in the report and are cheap to enable on failure, but a trace is strictly richer — it has the DOM snapshots, network, console, and action timeline you actually need to find a root cause. I enable screenshot/video on failure for a glance, but reach for the trace to actually debug.

## Gotchas & Anti-patterns

- **`trace: 'on'` everywhere** — large artifacts and slower runs. Use `'on-first-retry'` (or `retain-on-failure`).
- **Debugging flakes with `console.log` instead of the trace** — slow and lossy; the trace already has DOM/network/console at the failing moment.
- **Shipping raw codegen output** — brittle selectors, no assertions, no structure. Treat it as a draft.
- **Not uploading the trace as a CI artifact** — then you can't open it after the job ends. Publish `trace.zip`/the report (see [CI](./13-reporters-and-ci-integration.md)).
- **`page.pause()` left in committed code** — hangs CI waiting for manual input. Remove it before committing.
- **Reaching for screenshots when a trace would pinpoint the cause** — the before-snapshot in the trace shows DOM state the screenshot can't.

## Further Reading

- [Trace viewer](https://playwright.dev/docs/trace-viewer)
- [UI mode](https://playwright.dev/docs/test-ui-mode)
- [Debugging tests](https://playwright.dev/docs/debug)
- [Codegen](https://playwright.dev/docs/codegen)
