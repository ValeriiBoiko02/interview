# Locators

> Locators are the foundation of stable Playwright tests. Picking them well is what separates a suite that survives a redesign from one that breaks on every CSS tweak.

## TL;DR

- A **locator** is a lazy, re-resolved description of an element — not a captured handle. It re-queries the DOM every time you use it, which is what makes auto-waiting work.
- Prefer **user-facing** locators: `getByRole` first, then `getByLabel`/`getByPlaceholder`/`getByText`, then `getByTestId`. Avoid brittle CSS/XPath.
- **Strictness:** a locator that matches >1 element throws on action. That's a feature — it catches ambiguity early.
- Refine with **chaining** and **`filter()`**; disambiguate with `first()`/`nth()`/`getByRole({ name })`.
- Never use the deprecated `page.$`/ElementHandle for test logic — they don't auto-wait and capture a stale snapshot.

## Locators vs ElementHandles — the core distinction

```ts
// ✅ Locator: a description. Re-resolved on every action; auto-waits.
const submit = page.getByRole('button', { name: 'Submit' });
await submit.click();   // waits for it to exist, be visible, stable, enabled

// ❌ ElementHandle (legacy): a snapshot of one node at query time.
const handle = await page.$('button.submit'); // no auto-wait; stale if DOM re-renders
await handle?.click();                          // can throw on detached node
```

A locator holds *how to find* the element, re-running the query each time — so if React re-renders and swaps the node, the locator still finds the new one. An `ElementHandle` points at a specific DOM node captured then; after a re-render it can be detached and throw. **Use locators for everything.** `page.$`/`page.$$`/`$eval` are discouraged.

## The locator priority list (resilience, high → low)

| Priority | Locator | Why |
|---|---|---|
| 1 | `getByRole(role, { name })` | Mirrors how users and assistive tech perceive the page; survives styling/markup changes; doubles as an a11y check. |
| 2 | `getByLabel` / `getByPlaceholder` | Form controls by their visible label — stable and meaningful. |
| 3 | `getByText` / `getByAltText` / `getByTitle` | Good for non-interactive content; text can change with copy edits/i18n. |
| 4 | `getByTestId` | Explicit contract (`data-testid`) immune to copy/layout changes — but invisible to users, so use when semantics don't suffice. |
| 5 | CSS / XPath (`locator('.foo')`) | Last resort; couples tests to implementation details. |

```ts
page.getByRole('button', { name: 'Add to cart' });
page.getByRole('heading', { name: 'Checkout', level: 1 });
page.getByLabel('Email address');
page.getByPlaceholder('Search products');
page.getByText('Order confirmed', { exact: true });
page.getByTestId('cart-count');     // <div data-testid="cart-count">
```

**Why role-first?** It's the most decoupled from implementation: a button styled as a `<div role="button">` or restyled entirely is still `getByRole('button', { name })`. It also nudges you toward accessible markup. Reserve `data-testid` for cases where there's no good accessible handle (icon-only controls, ambiguous containers).

## Chaining, filtering, and disambiguation

```ts
// Scope within a region — read inside-out: the "Save" button within the toolbar.
await page.getByRole('toolbar').getByRole('button', { name: 'Save' }).click();

// filter() narrows a set by text or by a child locator
const row = page.getByRole('row').filter({ hasText: 'Invoice #42' });
await row.getByRole('button', { name: 'Pay' }).click();

// has / hasNot — filter rows that contain (or lack) another element
const paidRows = page.getByRole('row').filter({ has: page.getByText('Paid') });

// positional selection when items are genuinely identical
await page.getByRole('listitem').first().click();
await page.getByRole('listitem').nth(2).click();
await page.getByRole('listitem').last().click();

// count and iterate
const items = page.getByRole('listitem');
await expect(items).toHaveCount(3);
for (const item of await items.all()) {
  await expect(item).toBeVisible();
}
```

Prefer `filter({ hasText })` or a more specific `name` over `nth()` — positional indices are fragile when order changes. `nth()` is acceptable only when elements are truly interchangeable.

## Strictness mode

```ts
// If two buttons say "Delete", this THROWS rather than silently clicking the first:
await page.getByRole('button', { name: 'Delete' }).click();
// strict mode violation: resolved to 2 elements
```

Locator actions and most assertions are **strict**: matching multiple elements is an error. This catches ambiguous selectors at the point of failure instead of letting a test silently act on the wrong element. To act on one of many *intentionally*, be explicit: `.first()`, `.nth(i)`, or a narrower `filter`/`name`.

## Interview Q&A

**Q: What's the difference between a locator and an ElementHandle, and why does it matter?**
A: A locator is a lazy *description* re-resolved against the live DOM every time it's used; an ElementHandle is a reference to a specific node captured at query time. Because the locator re-queries, it auto-waits and survives re-renders — the node can be replaced and the locator still finds it. An ElementHandle goes stale/detached on re-render and doesn't auto-wait. Modern Playwright is locators-only; `page.$` is legacy.

**Q: What's your locator priority order and why?**
A: `getByRole` with an accessible name first, because it's how users/AT perceive the UI and is decoupled from styling and markup — and it implicitly checks accessibility. Then label/placeholder for inputs, then visible text, then `data-testid` when there's no semantic handle, and CSS/XPath only as a last resort. The goal is to bind tests to *behavior/contract*, not implementation details, so a redesign doesn't break them.

**Q: When is `data-testid` the right call over a role/text locator?**
A: When the element has no stable accessible name or the text is volatile (i18n, frequently edited copy, icon-only buttons), or when a container is genuinely ambiguous. `data-testid` is an explicit, intentional contract between dev and test. The trade-off is it's invisible to users so it doesn't validate accessibility — so it's a deliberate fallback, not the default.

**Q: What is strict mode and why is throwing on multiple matches a good thing?**
A: Locator actions resolve to exactly one element; matching several throws a strict-mode violation. It's good because the alternative — silently acting on the first match — hides bugs and produces tests that pass for the wrong reason. The throw forces you to write an unambiguous locator or explicitly pick (`first`/`nth`/`filter`), failing loudly at the real cause.

**Q: How do you reliably target one row in a dynamic table?**
A: Anchor on content, not position: `getByRole('row').filter({ hasText: 'Invoice #42' })`, then chain to the control inside it (`.getByRole('button', { name: 'Pay' })`). This survives reordering and pagination. `nth()` only when rows are truly identical and order is guaranteed.

## Gotchas & Anti-patterns

- **`page.$` / `$$` / `$eval` in test logic** — no auto-waiting, stale on re-render. Use locators.
- **`await locator.click()` on an ambiguous locator** — strict-mode throw. Narrow with `name`/`filter`, or pick explicitly with `first`/`nth`.
- **Brittle CSS like `div > div:nth-child(3) > .btn-primary`** — breaks on any layout change. Reach for role/label/testid.
- **Storing `await locator.all()` results and reusing them after the DOM changes** — those are snapshots; re-query instead.
- **Using `getByText` for interactive controls** — text changes with copy/i18n; use `getByRole(..., { name })`, which targets the accessible name.
- **`nth(0)` as a habit** — masks selector ambiguity. Prefer a specific match; reserve positional access for truly identical items.

## Further Reading

- [Locators](https://playwright.dev/docs/locators)
- [Other (legacy) selectors & why locators win](https://playwright.dev/docs/other-locators)
- [Accessibility / role locators](https://playwright.dev/docs/locators#locate-by-role)
- [`filter()`](https://playwright.dev/docs/locators#filtering-locators)
