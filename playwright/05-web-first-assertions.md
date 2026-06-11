# Web-first Assertions

> The auto-retrying `expect(locator)` family is what makes Playwright assertions resilient to async UIs. Knowing which assertions retry — and which don't — is the most common Playwright interview probe.

## TL;DR

- `await expect(locator).toBeVisible()` — **auto-retries** until it passes or `expect.timeout` (default 5s) elapses. This is the web-first family.
- `expect(value).toBe(...)` — a **one-shot** check on a plain value; no retry. Mixing the two up is the classic bug.
- `expect.soft` collects failures without aborting the test; `expect.poll` and `expect.toPass` retry arbitrary code.
- The defining gotcha: `await expect(locator).toBeVisible()` (retries) vs `expect(await locator.isVisible()).toBe(true)` (snapshot, races).
- Set per-assertion timeouts with `{ timeout }`; configure the default in `expect.timeout`.

## Auto-retrying vs one-shot — the core split

```ts
// ✅ Web-first: polls the DOM until the condition holds (or times out)
await expect(page.getByRole('alert')).toHaveText('Saved');
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();
await expect(page).toHaveURL('/dashboard');

// ❌ One-shot: evaluates the boolean ONCE, right now — no waiting, races the UI
expect(await page.getByRole('alert').isVisible()).toBe(true);
```

`expect(locator)` returns a matcher that *re-queries and re-checks* on an interval until it's satisfied or the timeout hits — so it naturally waits for async updates (a fetch resolving, a re-render). `expect(value)` receives an already-resolved value; there's nothing to retry. Whenever you find yourself writing `expect(await something.isX()).toBe(true)`, switch to the locator assertion.

## The assertion you'll reach for most

```ts
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toHaveText('Exact text');         // full text, retried
await expect(locator).toContainText('partial');
await expect(locator).toHaveValue('user@example.com');  // inputs
await expect(locator).toHaveCount(3);
await expect(locator).toBeEnabled();
await expect(locator).toBeChecked();
await expect(locator).toHaveAttribute('aria-expanded', 'true');
await expect(locator).toHaveClass(/active/);
await expect(page).toHaveTitle(/Dashboard/);
await expect(page).toHaveURL(/\/orders\/\d+/);
```

`toHaveText` with an **array** asserts on a list, retrying until the whole set matches — great for verifying rendered collections:

```ts
await expect(page.getByRole('listitem')).toHaveText(['Apples', 'Bananas', 'Cherries']);
```

## Soft assertions

```ts
await expect.soft(page.getByTestId('subtotal')).toHaveText('$42.00');
await expect.soft(page.getByTestId('tax')).toHaveText('$3.36');
await expect(page.getByTestId('total')).toHaveText('$45.36'); // hard: a failure here still aborts
```

`expect.soft` records a failure but lets the test continue, so one run surfaces *all* mismatches instead of stopping at the first. Use it to validate several independent facts about one screen. The test still ends up failed if any soft assertion failed. Don't use soft for preconditions that later steps depend on — those should be hard.

## Retrying arbitrary conditions: `poll` and `toPass`

```ts
// expect.poll — retry a function that returns a value, until the matcher passes
await expect.poll(async () => {
  const res = await page.request.get('/api/job/123');
  return (await res.json()).status;
}, { timeout: 10_000, intervals: [500, 1_000, 2_000] }).toBe('complete');

// expect.toPass — retry a whole block until it stops throwing
await expect(async () => {
  await page.getByRole('button', { name: 'Refresh' }).click();
  await expect(page.getByRole('row')).toHaveCount(10);
}).toPass({ timeout: 15_000 });
```

`expect.poll` is for polling a *value* (often an API/computed result). `expect.toPass` retries an arbitrary block of actions+assertions — handy for eventually-consistent flows where you may need to re-trigger something. Both are escape hatches: prefer a plain locator assertion when one fits, since they're clearer and cheaper.

## Timeouts

```ts
await expect(page.getByText('Report ready')).toBeVisible({ timeout: 30_000 }); // this one is slow
```

Per-assertion `{ timeout }` overrides the global `expect.timeout` ([setup](./01-setup-and-configuration.md)). Raise it for a legitimately slow expectation rather than inflating the global default (which slows down *every* failure). Note this is separate from the per-test `timeout`.

## Interview Q&A

**Q: What makes an assertion "web-first", and which ones are?**
A: Web-first assertions take a *locator* (or `page`) and auto-retry — they re-query the DOM on an interval until the condition holds or `expect.timeout` elapses. That's the whole `expect(locator).toBeVisible/toHaveText/toBeEnabled/...` family, plus `expect(page).toHaveURL/toHaveTitle`. Assertions on plain values (`expect(number).toBe(...)`) are one-shot and don't retry.

**Q: What's wrong with `expect(await locator.isVisible()).toBe(true)`?**
A: `isVisible()` resolves to a boolean *at that instant* with no waiting, so you're asserting a snapshot — if the element appears 50ms later, the test fails or flakes. `await expect(locator).toBeVisible()` retries until it's visible (or times out). Same intent, but one races the UI and the other synchronizes with it. This is the canonical Playwright assertion mistake.

**Q: When would you use `expect.soft`?**
A: When verifying several *independent* facts on one screen and you want all failures in a single run — e.g. asserting subtotal, tax, and shipping. Soft assertions record failures and continue, then the test ends failed if any failed. I keep anything a later step depends on as a hard assertion so the test stops before doing something meaningless.

**Q: `expect.poll` vs `expect.toPass` — when each?**
A: `expect.poll` when I can express the check as "compute a value, match it" — e.g. poll an API until `status === 'complete'`. `expect.toPass` when I need to retry a *block* of actions and assertions together, e.g. click Refresh then assert the row count, for eventually-consistent UIs. Both are fallbacks; a single locator assertion is preferable when it suffices.

**Q: How do timeouts interact here?**
A: Each web-first assertion has its own retry budget (`expect.timeout`, default 5s), overridable per call with `{ timeout }`. That's distinct from the per-test `timeout` (default 30s) and per-action `actionTimeout`. A test can blow its 30s budget from many slow assertions without any single assertion hitting its 5s limit.

## Gotchas & Anti-patterns

- **`expect(await locator.isVisible()).toBe(true)`** — non-retrying snapshot; races async UI. Use `await expect(locator).toBeVisible()`.
- **Forgetting `await` on a web-first assertion** — the promise never settles, the assertion effectively no-ops, and the test passes for the wrong reason. Always `await expect(...)`.
- **`toHaveText` vs `toContainText`** — `toHaveText` is a full-string match (whitespace-normalized); use `toContainText` for substrings. Mixing them up causes confusing failures.
- **Inflating global `expect.timeout` to fix one slow check** — slows every failure. Scope a longer `{ timeout }` to the one assertion.
- **Using `expect.poll`/`toPass` where a locator assertion fits** — more code, less clarity, no auto-wait integration with the trace. Reach for them only for genuinely arbitrary/eventual conditions.
- **Soft assertions on preconditions** — if later steps depend on it, a soft failure lets the test barrel on into garbage. Keep those hard.

## Further Reading

- [Assertions](https://playwright.dev/docs/test-assertions)
- [Auto-retrying assertions list](https://playwright.dev/docs/test-assertions#auto-retrying-assertions)
- [`expect.poll` / `expect.toPass`](https://playwright.dev/docs/test-assertions#expectpoll)
- [Soft assertions](https://playwright.dev/docs/test-assertions#soft-assertions)
