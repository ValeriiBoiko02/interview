# Page Object Model & Test Architecture

> How to structure a suite so it survives UI churn and stays readable at scale. POMs are the well-known pattern; the senior view is knowing when they help, how fixtures supersede the boilerplate, and where over-abstraction hurts.

## TL;DR

- A **Page Object** encapsulates the locators and interactions for a page/region behind intent-revealing methods, so specs read as behavior, not selectors.
- Build POMs on **locators** (store them as fields/getters), never on captured ElementHandles.
- **Inject POMs through fixtures** ([fixtures](./06-fixtures.md)) instead of `new`-ing them in every test.
- Prefer **small component objects** (a header, a data grid, a modal) over one giant page class.
- The real debate: keep assertions out of POMs (POM = interactions, spec = assertions) vs. allow expectation helpers. Have a defensible position.
- Over-abstraction (deep inheritance, a method per click) is a common failure mode — POMs should remove duplication, not add ceremony.

## A POM built on locators

```ts
// pages/login.page.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly email: Locator;
  readonly password: Locator;
  readonly submit: Locator;
  readonly error: Locator;

  constructor(page: Page) {
    this.page = page;
    this.email = page.getByLabel('Email');
    this.password = page.getByLabel('Password');
    this.submit = page.getByRole('button', { name: 'Sign in' });
    this.error = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  // intent-revealing action — the spec says WHAT, the POM knows HOW
  async login(email: string, password: string) {
    await this.email.fill(email);
    await this.password.fill(password);
    await this.submit.click();
  }
}
```

Locators are assigned in the constructor as fields. Because locators are lazy ([locators](./03-locators.md)), assigning them up front is safe — they don't query until used. Methods express user intent (`login`), keeping selector details in one place so a markup change touches one file.

## Inject the POM via a fixture

```ts
// fixtures.ts
import { test as base } from '@playwright/test';
import { LoginPage } from './pages/login.page';
import { DashboardPage } from './pages/dashboard.page';

export const test = base.extend<{ loginPage: LoginPage; dashboard: DashboardPage }>({
  loginPage: async ({ page }, use) => use(new LoginPage(page)),
  dashboard: async ({ page }, use) => use(new DashboardPage(page)),
});
export { expect } from '@playwright/test';
```

```ts
// login.spec.ts — clean, no `new`, no selectors
import { test, expect } from '../fixtures';

test('rejects bad credentials', async ({ loginPage }) => {
  await loginPage.goto();
  await loginPage.login('user@example.com', 'wrong');
  await expect(loginPage.error).toHaveText('Invalid credentials');
});
```

This is the modern idiom: fixtures handle construction/wiring, specs read as scenarios. It also makes POMs composable — `authedPage` can build on `loginPage`.

## Component objects beat one giant page class

Model reusable regions, not just full pages:

```ts
// components/data-grid.component.ts
export class DataGrid {
  constructor(private readonly root: Locator) {}

  row(text: string): Locator {
    return this.root.getByRole('row').filter({ hasText: text });
  }
  async sortBy(column: string) {
    await this.root.getByRole('columnheader', { name: column }).click();
  }
}
```

A shared header, modal, or table appears on many pages; a component object scoped to a `root` locator is reusable across them and keeps each class small. Composition (a page *has* a `DataGrid`) beats a deep page-class inheritance tree.

## The assertions-in-POM debate

Two defensible positions — pick one and apply it consistently:

| Approach | Pros | Cons |
|---|---|---|
| **POM = interactions only; assertions in the spec** | Clear separation; specs document expected behavior; POMs stay reusable across tests with different expectations. | Specs are slightly longer; some expectation logic repeats. |
| **POM exposes expectation helpers** (e.g. `expectLoginError(msg)`) | DRY for assertions repeated across many tests; reads fluently. | Risks hiding what's verified; couples POM to a specific expectation. |

A common senior compromise: POMs expose **locators and actions**; specs own assertions — but allow a few well-named expectation helpers for genuinely repeated, complex checks. The anti-pattern is burying *all* assertions inside POM methods so a spec is just `await page.doEverything()` and you can't tell what's verified.

## Folder structure that scales

```
tests/
  fixtures.ts            # extended `test` wiring POMs + sessions
  pages/                 # one file per page object
    login.page.ts
    dashboard.page.ts
  components/            # reusable region objects
    header.component.ts
    data-grid.component.ts
  specs/                 # the actual tests, grouped by feature
    auth/login.spec.ts
    orders/checkout.spec.ts
  data/                  # test data factories / fixtures
  utils/                 # pure helpers (no Playwright state)
```

Group specs by feature, not by type, once the suite grows. Keep pure helpers (data builders, formatters) free of Playwright objects so they're unit-testable and reusable.

## Interview Q&A

**Q: Why use the Page Object Model, and what's the senior critique of it?**
A: POMs centralize selectors and interactions so a UI change touches one file and specs read as behavior, not CSS. The critique: teams over-apply it — deep inheritance hierarchies, a wrapper method for every single click, assertions hidden inside, so the abstraction adds more ceremony than it removes. The goal is to eliminate duplication and express intent; if a POM does neither, it's just indirection. Favor small component objects and composition over a monolithic page class.

**Q: How do POMs and fixtures relate — competing or complementary?**
A: Complementary. POMs encapsulate page interaction; fixtures handle their construction and lifecycle and inject them into tests. Instead of `new LoginPage(page)` in every test, a `loginPage` fixture provides a ready instance, and POMs can build on each other (an `authedPage` fixture using the login flow). Fixtures are the DI container; POMs are the objects it injects.

**Q: Should assertions live in page objects?**
A: My default: POMs expose locators and actions, specs own assertions, so it's obvious from the test what behavior is being verified and POMs stay reusable for tests with different expectations. I'll allow a few named expectation helpers for complex checks repeated across many tests, but I avoid the pattern where a spec is one opaque method call and you can't see what's asserted.

**Q: How do you keep POMs from going stale or bloated?**
A: Build them on locators (lazy, re-resolved) not handles; prefer component objects scoped to a root locator over one giant class; use composition over inheritance; keep pure logic in framework-free helpers; and inject via fixtures so wiring lives in one place. Group specs by feature. The test of a good POM is whether a redesign of one screen touches one file.

**Q: When is a POM overkill?**
A: For a tiny suite, a one-off page, or a flow used in a single test, a POM can be more indirection than value — a few inline locators are clearer. POMs earn their keep when the same page/region is exercised by many tests and selectors would otherwise be duplicated.

## Gotchas & Anti-patterns

- **Storing `ElementHandle`s in POM fields** — they go stale on re-render. Store **locators**; they re-resolve.
- **`new LoginPage(page)` in every test** — boilerplate that drifts. Inject through a fixture.
- **One 1,000-line page class** — split into component objects; compose them.
- **Deep inheritance trees of base page classes** — fragile and hard to follow. Prefer composition.
- **Hiding all assertions inside POM methods** — specs become opaque (`await page.checkout()` with no visible expectations). Keep verification in the spec (or in clearly-named expectation helpers).
- **A wrapper method per single action (`clickSubmit()`)** — adds ceremony without removing duplication. Methods should capture *intent* (`login(email, pw)`), not 1:1 wrap clicks.
- **Putting waits/`waitForTimeout` in POMs** — rely on auto-waiting actions and web-first assertions instead ([actions](./04-actions-and-auto-waiting.md)).

## Further Reading

- [Page Object Models](https://playwright.dev/docs/pom)
- [Fixtures](https://playwright.dev/docs/test-fixtures) (the modern way to wire POMs)
- [Locators](./03-locators.md) · [Fixtures](./06-fixtures.md)
