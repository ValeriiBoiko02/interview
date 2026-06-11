# Flakiness & Stability

> A cross-cutting playbook: the taxonomy of why tests flake and the concrete fix for each. Flakiness is *the* defining challenge of a real automation suite, so this is the highest-signal interview topic — expect to be probed hard.

## TL;DR

- A **flaky test** passes and fails without code changes — it's nondeterministic. Flaky tests are worse than no tests: they erode trust until people ignore red.
- Most flakiness has a root cause in one of: **bad waits**, **test interdependence/shared state**, **uncontrolled network**, **animations/timing**, or **time/locale/data nondeterminism**.
- Playwright's auto-waiting ([actions](./04-actions-and-auto-waiting.md)) and web-first assertions ([assertions](./05-web-first-assertions.md)) eliminate the biggest category by design — *if* you use them.
- **Retries are a detector, not a cure** ([parallelism](./11-parallelism-retries-and-sharding.md)). A test that only passes on retry is a defect.
- The trace viewer ([debugging](./12-debugging-and-trace-viewer.md)) turns "flaky in CI" into a concrete root cause.

## The taxonomy — cause → fix

| Cause | Symptom | Fix |
|---|---|---|
| **Manual sleeps / racing reads** | `waitForTimeout`, `isVisible()`-then-act; passes fast, fails slow | Web-first assertions + auto-waiting actions; wait on *conditions*, never durations. |
| **Test interdependence** | Passes alone, fails in parallel / different order | Each test owns its data (unique emails, API-seeded records); worker-scoped fixtures; never depend on order. |
| **Shared external state** | Same DB row / account mutated by concurrent tests | Generate unique data per test; isolate via fixtures; clean up via API teardown. |
| **Uncontrolled network** | Real backend slow/flaky/rate-limited | Mock the dependency ([network](./08-network-mocking-and-interception.md)); assert on UI state, not arbitrary timing. |
| **Animations / transitions** | Click "misses"; element moving | Auto-waiting handles most (waits for stability); disable animations in test env if needed. |
| **Time / locale / timezone** | Fails overnight, in CI's TZ, or on month boundaries | Freeze the clock (`page.clock`); pin locale/timezone in context options; avoid `new Date()` assumptions. |
| **Random / ordered data** | Assertion on volatile order or random values | Sort/normalize before asserting; seed randomness; assert on stable identifiers. |
| **Over-parallelism** | Flakiness rises with worker count | Match workers to CI cores; reduce contention. |
| **Auth/session expiry** | Mass failures after a while | Regenerate `storageState` per run; don't cache sessions ([auth](./09-authentication-and-storage-state.md)). |

## The single biggest source: synchronization done wrong

```ts
// ❌ Flaky: guesses a duration, then reads a non-waiting boolean
await page.waitForTimeout(2000);
expect(await page.getByRole('alert').isVisible()).toBe(true);

// ✅ Stable: synchronizes on the actual condition, auto-retried
await expect(page.getByRole('alert')).toBeVisible();
```

Selenium-era habits (`sleep`, then assert) are the number-one cause of flakiness, and Playwright removes them *if you let it*. Auto-waiting actions wait for actionability; web-first assertions poll until true. The rule: **never synchronize on time; synchronize on state.** Almost every `waitForTimeout` in a suite is a latent flake.

## Controlling nondeterministic inputs

```ts
// Freeze time so date-dependent UI is deterministic
await page.clock.install({ time: new Date('2026-06-10T12:00:00Z') });
await page.clock.fastForward('01:00');   // advance an hour without real waiting

// Pin locale & timezone at the context level (config `use` or test.use)
test.use({ locale: 'en-US', timezoneId: 'UTC' });

// Mock a flaky/slow dependency so the test is deterministic
await page.route('**/api/quote', r =>
  r.fulfill({ json: { price: 100 } }));
```

Anything the test doesn't control — wall-clock time, timezone, locale, randomness, third-party APIs — is a flakiness vector. Pin or mock each one. A test should fail only when the product is broken, never because it ran at 23:59 or in a different region.

## A process for killing a flake

1. **Reproduce deterministically:** run the test many times (`--repeat-each=20`) and/or with high parallelism (`--workers=...`) to surface it.
2. **Isolate the variable:** if `--workers=1` makes it pass, it's interdependence/shared state. If it's time-of-day, it's a clock/locale issue.
3. **Read the trace:** open the failing run's trace; the before-snapshot at the red action usually shows the cause (overlay, spinner, not-rendered).
4. **Fix the root cause**, don't pad timeouts or add retries.
5. **Verify** by re-running `--repeat-each` until it's stably green.
6. **Track flaky rate** as a metric; a rising count is tech debt, not noise.

```bash
npx playwright test flaky.spec.ts --repeat-each=20 --workers=4   # try to reproduce
npx playwright test flaky.spec.ts --repeat-each=50 --workers=1   # isolate parallelism vs logic
```

## Retries: detector, not cure

Retries ([parallelism](./11-parallelism-retries-and-sharding.md)) keep an occasional infra blip from failing the build and *flag* tests that pass on retry as flaky. That flag is a to-do, not a resolution. A suite where many tests rely on retries to go green has hidden nondeterminism and is one bad day from a credibility collapse. Keep `retries: 0` locally so flakes are visible; treat the CI flaky list as a defect backlog.

## Interview Q&A

**Q: What is a flaky test and why is it dangerous?**
A: One that passes or fails nondeterministically with no code change. It's dangerous because it destroys trust: once a suite cries wolf, people start re-running until green or ignoring failures — and then a real regression slips through. A flaky test is arguably worse than no test, because it carries the cost of a test plus the cost of eroded confidence.

**Q: What are the main causes of flakiness and how does Playwright help?**
A: The big buckets are bad synchronization (sleeps/racing reads), test interdependence/shared state, uncontrolled network, animation/timing, and time/locale/data nondeterminism. Playwright eliminates the largest bucket by design — auto-waiting actions and auto-retrying web-first assertions synchronize on state, not time — so you rarely need explicit waits. The rest you fix with isolation (unique data, fixtures), network mocking, and pinning the clock/locale.

**Q: How do you actually debug and fix a flaky test?**
A: Reproduce it with `--repeat-each` and varied `--workers`; isolate whether it's parallelism (passes at `--workers=1`) or timing (fails at certain times). Open the trace and look at the before-snapshot at the failing action — usually an overlay, spinner, or unrendered element. Fix the root cause (right locator, web-first assertion, own your data, mock the dependency), then re-run `--repeat-each` to confirm stability. I never "fix" it by raising timeouts or adding retries.

**Q: Are retries a fix for flakiness?**
A: No. Retries detect flakiness (a pass-on-retry is reported as flaky) and absorb rare infra blips, but they don't address nondeterminism — they hide it. I allow a small retry count in CI for resilience and keep it at 0 locally so flakes surface, and I treat the flaky list as a backlog of defects to fix, not a steady state.

**Q: How do you make date/time-dependent tests stable?**
A: Freeze time with `page.clock.install(...)` and advance it deterministically with `fastForward`, and pin `locale`/`timezoneId` in the context. That removes the "fails overnight / in CI's timezone / on a month boundary" class of flakes. More generally: anything the test doesn't control — time, locale, randomness, third-party APIs — gets pinned or mocked.

**Q: How do you prevent flakiness on a team, not just fix it?**
A: Bake the right defaults in: web-first assertions only (lint/ban `waitForTimeout`), data isolation by default (API-seeded unique data, no order dependence), `trace: 'on-first-retry'` so every failure is diagnosable, `forbidOnly` and `retries:0` locally, and a tracked flaky-rate metric so regressions in stability are visible. Code review catches sleeps and shared-state patterns before they land.

## Gotchas & Anti-patterns

- **`waitForTimeout` as synchronization** — the canonical flake. Wait on conditions; use web-first assertions.
- **Raising timeouts to "stabilize"** — slows the suite and hides the real cause. Fix the wait/locator.
- **Adding retries until it's green** — masks nondeterminism; retries are a signal, not a fix.
- **Tests that depend on order/leftover state** — break under parallelism. Own your data; isolate.
- **Asserting on volatile order/random values/timestamps** — normalize, seed, freeze the clock, assert on stable IDs.
- **Hitting a live third-party API in every test** — imports its flakiness. Mock it; keep a thin real-integration layer.
- **Ignoring the flaky list** — it's a defect backlog. A normalized flaky rate that's allowed to climb ends in a suite nobody trusts.

## Further Reading

- [Best Practices](https://playwright.dev/docs/best-practices)
- [Clock (freezing time)](https://playwright.dev/docs/clock)
- [Auto-waiting](https://playwright.dev/docs/actionability) · [Web-first assertions](./05-web-first-assertions.md) · [Trace viewer](./12-debugging-and-trace-viewer.md)
