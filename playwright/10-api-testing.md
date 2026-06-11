# API Testing

> Playwright can drive HTTP directly, not just the browser. That powers standalone API tests and — more importantly for E2E suites — fast, reliable setup/teardown that bypasses the UI.

## TL;DR

- The `request` fixture gives you an `APIRequestContext` for `get`/`post`/`put`/`delete` — no browser needed.
- Two big uses: **standalone API tests** and **hybrid** tests that seed/clean data via API but verify through the UI.
- API setup is **faster and less flaky** than clicking through forms to create preconditions.
- Reuse browser auth in API calls (and vice versa) via shared `storageState`.
- API tests in an E2E suite are for *enabling* UI tests and smoke-checking endpoints — not a replacement for a real contract/integration test layer.

## Standalone API tests

```ts
import { test, expect } from '@playwright/test';

test('GET /api/orders returns the user\'s orders', async ({ request }) => {
  const res = await request.get('/api/orders');     // baseURL-relative
  expect(res.status()).toBe(200);
  const body = await res.json();
  expect(body.orders).toHaveLength(3);
  expect(body.orders[0]).toMatchObject({ id: expect.any(Number), status: 'shipped' });
});

test('POST /api/orders creates an order', async ({ request }) => {
  const res = await request.post('/api/orders', {
    data: { productId: 42, quantity: 2 },
    headers: { 'content-type': 'application/json' },
  });
  expect(res.status()).toBe(201);
  expect(await res.json()).toMatchObject({ productId: 42, quantity: 2 });
});
```

The `request` fixture respects `baseURL` and config-level `extraHTTPHeaders`. Note these are *resolved values*, so assertions on them are plain `expect(...)`, not the auto-retrying web-first family ([assertions](./05-web-first-assertions.md)).

## Hybrid: seed/clean via API, verify via UI (the high-value pattern)

```ts
test('a created order shows up in the dashboard', async ({ page, request }) => {
  // Arrange — create the precondition over HTTP (fast, deterministic)
  const res = await request.post('/api/orders', { data: { productId: 42, quantity: 1 } });
  const { id } = await res.json();

  // Act + Assert — verify through the real UI
  await page.goto('/orders');
  await expect(page.getByRole('row').filter({ hasText: `#${id}` })).toBeVisible();

  // Teardown — clean up via API so the test is repeatable and isolated
  await request.delete(`/api/orders/${id}`);
});
```

This is where API capability pays off most in an E2E suite: building state through the UI is slow and turns unrelated forms into flakiness sources. Create the data over HTTP, then test only the behavior you actually care about in the browser. Clean up via API so the test stays isolated and re-runnable.

## A standalone API context (outside the runner / shared base)

```ts
import { request } from '@playwright/test';

const api = await request.newContext({
  baseURL: 'https://staging.example.com',
  extraHTTPHeaders: { Authorization: `Bearer ${process.env.API_TOKEN}` },
});
const res = await api.get('/api/health');
expect(res.ok()).toBeTruthy();
await api.dispose();   // free resources when done
```

`request.newContext()` is the manual form (e.g. in a fixture or `globalSetup`). It's also how you share auth headers across many API calls.

## Sharing auth between API and UI

```ts
// Authenticate over the API once, persist, and reuse for BOTH api and browser:
setup('api auth', async ({ request }) => {
  await request.post('/api/login', { data: { email: process.env.TEST_USER, password: process.env.TEST_PASS } });
  await request.storageState({ path: 'playwright/.auth/user.json' });
});
// then: use that storageState for the browser projects (see Auth topic) AND
// new APIRequestContexts created from a context inherit its cookies.
```

Because cookies live in the storage state, the same session works for `page` and for `request` calls made from that context — log in once, use everywhere. See [auth & storage state](./09-authentication-and-storage-state.md).

## Where API testing belongs in an E2E suite

| Good fit in an E2E/Playwright suite | Belongs elsewhere |
|---|---|
| Seeding/cleaning data to support UI tests. | Exhaustive endpoint coverage and contract testing — better in a dedicated API/contract suite (e.g. Pact, supertest) closer to the service. |
| Smoke-checking critical endpoints (health, auth, key reads). | Heavy validation/business-logic testing of the backend — unit/integration tests own that. |
| Asserting the UI and API agree (catch front-end/back-end drift). | Performance/load testing — separate tooling. |

Playwright's HTTP client is excellent and convenient, but a sprawling API regression suite usually lives closer to the service it tests. In the browser suite, lean on API mostly to *enable* and *anchor* UI tests.

## Interview Q&A

**Q: How does Playwright do API testing, and when do you use it?**
A: The `request` fixture exposes an `APIRequestContext` with `get`/`post`/`put`/`delete`, honoring `baseURL` and configured headers — no browser involved. I use it standalone to smoke-check endpoints, and far more often in *hybrid* tests: create preconditions and tear down data over HTTP while verifying behavior through the UI. That makes UI tests faster and less flaky than building state by clicking.

**Q: Why seed test data via API instead of through the UI?**
A: Speed and determinism. Clicking through a multi-step form to create a precondition is slow, couples the test to unrelated UI, and adds flakiness. A single API call sets up the exact state, so the test focuses on the behavior under test. I also clean up via API afterward to keep tests isolated and repeatable.

**Q: Are API assertions auto-retrying like UI ones?**
A: No. `request` returns resolved response objects, so you assert with plain `expect(res.status()).toBe(200)` — one-shot, no retry. The auto-retrying web-first family is specifically for locators/page. If I need to poll an endpoint until it's ready, I wrap it in `expect.poll`.

**Q: How do you share authentication between API and browser?**
A: Authenticate once (often over the API, which is fastest), save `storageState`, and reuse it for both: browser projects load it via `use: { storageState }`, and API contexts created from a browser context inherit its cookies. One login, used by both layers.

**Q: Should your whole API test suite live in Playwright?**
A: For an E2E project, no — I keep API usage focused on enabling UI tests and a thin layer of endpoint smoke/contract-drift checks. Exhaustive endpoint and business-logic coverage belongs in a dedicated API/contract/integration suite closer to the service, which is faster to run and owned alongside the backend. Right tool, right layer.

## Gotchas & Anti-patterns

- **Expecting API assertions to auto-retry** — they don't; use `expect.poll` for eventual readiness.
- **Building UI preconditions by clicking** when an API call would do — slow and flaky. Seed via `request`.
- **Not cleaning up created data** — tests leak state and stop being repeatable/parallel-safe. Delete via API in teardown.
- **Duplicating a full backend regression suite in Playwright** — slow browser-test infra for something an API/contract suite does better. Keep it thin and purposeful.
- **Forgetting `dispose()` on a manually created `request.newContext()`** — leaks resources. Test-scoped `request` fixture is auto-disposed; manual contexts aren't.
- **Hardcoding tokens** — pull from env/secrets, mirror the [auth](./09-authentication-and-storage-state.md) practices.

## Further Reading

- [API testing](https://playwright.dev/docs/api-testing)
- [`APIRequestContext`](https://playwright.dev/docs/api/class-apirequestcontext)
- [`expect.poll`](https://playwright.dev/docs/test-assertions#expectpoll) for eventual-consistency
