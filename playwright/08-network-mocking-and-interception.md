# Network Mocking & Interception

> Controlling the network lets you test edge cases (errors, empty states, slow responses) deterministically and decouple front-end tests from a flaky backend. Knowing *when* to mock is as important as *how*.

## TL;DR

- `page.route(url, handler)` intercepts matching requests; the handler decides: `fulfill` (fake response), `abort` (fail it), or `continue` (let through, optionally modified).
- `context.route(...)` applies to every page in the context; `page.route` is per page.
- **HAR record/replay** (`routeFromHAR`) captures real traffic once and replays it — mocking without hand-writing fixtures.
- `page.waitForResponse(...)` synchronizes on a specific network event when you need to.
- Mock to make tests **fast, deterministic, and able to hit edge cases**; hit the real backend for true end-to-end confidence. It's a deliberate trade-off, not a default.

## Intercepting and faking responses

```ts
test('renders the empty state when there are no orders', async ({ page }) => {
  await page.route('**/api/orders', async route => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify({ orders: [] }),
    });
  });

  await page.goto('/orders');
  await expect(page.getByText('You have no orders yet')).toBeVisible();
});
```

Set up the route **before** the navigation/action that triggers the request. `route.fulfill` returns your fake response and the real request never goes out — perfect for empty states, specific data shapes, or responses your backend can't easily produce on demand.

## abort, continue, and modifying traffic

```ts
// Simulate a server error to test error handling
await page.route('**/api/checkout', route =>
  route.fulfill({ status: 500, body: 'Internal Server Error' }));

// Block third-party noise (analytics, ads, fonts) to speed up and stabilize
await page.route(/(google-analytics|doubleclick|hotjar)\.com/, route => route.abort());

// Let it through but tweak the request (add a header, change the payload)
await page.route('**/api/**', async route => {
  const headers = { ...route.request().headers(), 'x-test-run': 'true' };
  await route.continue({ headers });
});

// Inspect the real response, then fulfill with a modified version
await page.route('**/api/profile', async route => {
  const response = await route.fetch();        // perform the real request
  const json = await response.json();
  json.featureFlags.newDashboard = true;       // flip a flag for this test
  await route.fulfill({ response, json });
});
```

| Verb | Effect | Use for |
|---|---|---|
| `fulfill` | Return a fabricated response. | Mock data, error codes, empty/edge states. |
| `abort` | Fail the request (network error). | Offline/failure simulation; blocking 3rd-party noise. |
| `continue` | Forward it, optionally modifying method/headers/postData. | Injecting headers, rewriting requests. |
| `route.fetch()` + `fulfill({ response })` | Get the real response, then patch it. | Tweaking one field of a real payload (e.g. a feature flag). |

## HAR record & replay

```ts
// Record once against the real backend (writes network.har):
//   npx playwright open --save-har=network.har --save-har-glob="**/api/**" http://localhost:3000

// Replay in tests — no live backend needed:
await page.routeFromHAR('network.har', {
  url: '**/api/**',
  update: false,                 // true = re-record/refresh the HAR
});
```

`routeFromHAR` replays previously-recorded traffic, giving you realistic mocks without hand-authoring JSON. Re-record (`update: true`) when the API changes. The trade-off vs hand-written mocks: HARs are realistic and cheap to capture but can drift silently from the live API and are bulky to review in diffs.

## Synchronizing on the network

```ts
// Wait for a specific response triggered by an action (parallelize the wait + the click)
const [response] = await Promise.all([
  page.waitForResponse(r => r.url().includes('/api/search') && r.ok()),
  page.getByRole('button', { name: 'Search' }).click(),
]);
expect(response.status()).toBe(200);
```

Set up `waitForResponse` *before* the action that triggers it (hence `Promise.all`). Use it when you must assert on the response or genuinely need to wait for a backend round-trip — but prefer asserting on the resulting **UI state** ([web-first assertions](./05-web-first-assertions.md)), which is usually clearer and less brittle than waiting on the wire.

## When to mock vs hit the real backend

| Mock the network | Use the real backend |
|---|---|
| Edge cases the backend can't easily produce (500s, timeouts, empty/huge datasets). | True end-to-end confidence that the integration actually works. |
| Deterministic, fast front-end tests independent of backend availability. | Contract validation across the front-end/back-end boundary. |
| Isolating a flaky/slow/shared third-party dependency. | Smoke/critical-path suites where realism matters most. |

A balanced strategy: mock in the bulk of UI tests for speed and determinism, and keep a smaller real-backend E2E layer for integration confidence. Be explicit about which layer a given test belongs to. Mocks that drift from the real API give false confidence — pair heavy mocking with [API/contract tests](./10-api-testing.md).

## Interview Q&A

**Q: How does request interception work in Playwright?**
A: `page.route(pattern, handler)` (or `context.route` for all pages) registers a handler for matching requests. In the handler you choose `fulfill` to return a fake response, `abort` to fail it, or `continue` to forward it (optionally modifying method/headers/body). You can also `route.fetch()` the real response and then `fulfill` a patched version. The route must be registered before the request is made.

**Q: When should you mock the network, and what's the risk?**
A: Mock to make tests deterministic and fast, to exercise edge cases the backend can't easily produce (errors, empty states, huge lists), and to isolate flaky third-party calls. The risk is divergence: a mock can keep passing after the real API changes shape, giving false confidence. I mitigate by keeping a real-backend E2E layer and/or contract tests, and by refreshing HAR recordings when the API moves.

**Q: `page.route` vs `context.route`?**
A: `page.route` applies to a single page; `context.route` applies to every page/popup in the browser context. Use context-level for cross-cutting rules (blocking analytics for the whole session, a shared auth header); page-level for test-specific mocks.

**Q: HAR replay vs hand-written mocks — trade-offs?**
A: HAR replay (`routeFromHAR`) captures real traffic once and replays it — realistic and low-effort, great for complex APIs. Downsides: HAR files are large, awkward to review in diffs, and drift silently when the API changes (you must re-record). Hand-written mocks are explicit and reviewable and pin exactly the shape you care about, but they're more work and can also drift. I use HAR for breadth/realism and targeted `fulfill` mocks for specific edge cases.

**Q: How do you wait for a network call, and should you?**
A: `page.waitForResponse(predicate)` set up before the triggering action (via `Promise.all`) when I need to assert on the response itself. But usually I prefer to assert on the resulting UI state, because that's what the user observes and it's resilient to incidental network details. Waiting on the wire is a means, not the goal.

## Gotchas & Anti-patterns

- **Registering the route after triggering the request** — the real request already fired; the mock never applies. Set up `route`/`waitForResponse` first.
- **Over-broad patterns (`**/*`) that swallow unrelated requests** — including assets/HTML, breaking the page. Scope patterns to the API paths you mean.
- **Mocks drifting from the real API** — passing tests, broken product. Re-record HARs, add contract/API tests, and keep a real-backend layer.
- **`route.continue()` without un-routing for later requests** — handlers persist for the context/page lifetime; use `page.unroute` if a route should stop applying mid-test.
- **Using `waitForResponse` where a UI assertion would do** — couples the test to network internals. Prefer asserting the visible result.
- **Committing huge HAR files without review** — they hide secrets and bloat the repo; scrub and scope them with `--save-har-glob`.

## Further Reading

- [Network](https://playwright.dev/docs/network)
- [Mock APIs](https://playwright.dev/docs/mock)
- [`routeFromHAR` / HAR replay](https://playwright.dev/docs/mock#mocking-with-har-files)
