# Actions & Auto-waiting

> Playwright's auto-waiting is the single biggest reason its tests are less flaky than Selenium's. Understanding *what* it waits for — and why manual sleeps are an anti-pattern — is core senior knowledge.

## TL;DR

- Before every action, Playwright runs **actionability checks** and waits (up to a timeout) until they pass — no manual waits needed.
- The checks: element is **attached**, **visible**, **stable** (not animating), **receives events** (not covered), and **enabled** (for inputs/buttons).
- This is the **web-first** model: you describe the action; Playwright waits for the precondition. `waitForTimeout(ms)` as a sync mechanism is the cardinal anti-pattern.
- `force: true` skips the actionability checks — an escape hatch that usually hides a real bug.
- Pair auto-waiting actions with auto-retrying assertions ([web-first assertions](./05-web-first-assertions.md)) and you rarely need an explicit wait.

## What "actionability" means

Before `click()`, `fill()`, etc., Playwright auto-waits until the target passes all relevant checks:

| Check | Meaning | Applies to |
|---|---|---|
| **Attached** | Element is in the DOM. | all |
| **Visible** | Non-empty box, not `display:none`/`visibility:hidden`/`opacity:0`. | all |
| **Stable** | Bounding box unchanged across two animation frames (not mid-transition). | actions requiring a hit point |
| **Receives events** | It's the actual hit target at the action point (not behind an overlay/modal). | click, hover, tap |
| **Enabled** | Not `disabled`. | form controls |
| **Editable** | Enabled and not `readonly`. | `fill`, `type` |

If any check fails, Playwright keeps retrying until it passes or the action timeout elapses — then it throws with a clear reason (e.g. "element is not visible" / "intercepts pointer events"). That message is your debugging starting point.

## The web-first model

```ts
// You DON'T do this:
await page.waitForTimeout(2000);        // ❌ guessing; slow when fast, flaky when slow
await page.click('#submit');

// You DO this — the click waits for #submit to be actionable:
await page.getByRole('button', { name: 'Submit' }).click();   // ✅ waits intrinsically
```

The mental shift from Selenium: you don't *synchronize, then act*. The action carries its own synchronization. The only time you wait explicitly is for a condition Playwright can't infer — and even then you use a *condition*, not a fixed sleep:

```ts
// Wait for a real condition, not a guessed duration:
await page.waitForURL('/dashboard');
await page.waitForResponse(r => r.url().includes('/api/orders') && r.ok());
await expect(page.getByRole('status')).toHaveText('Saved');   // auto-retries — usually all you need
```

## Common actions

```ts
await page.getByRole('button', { name: 'Save' }).click();
await page.getByLabel('Email').fill('user@example.com');     // clears + sets in one shot
await page.getByLabel('Bio').clear();
await page.getByRole('checkbox', { name: 'Subscribe' }).check();
await page.getByRole('combobox', { name: 'Country' }).selectOption('US');
await page.getByRole('button', { name: 'Menu' }).hover();
await page.getByLabel('Avatar').setInputFiles('avatar.png');
await page.getByRole('textbox').press('Enter');
await page.getByText('Drag me').dragTo(page.getByTestId('dropzone'));
```

Prefer semantic actions (`check`, `selectOption`, `setInputFiles`) over simulating low-level events — they include the right actionability checks and reflect user intent.

### `fill` vs `pressSequentially`

`fill()` sets the value directly (fast, atomic) and fires the right input events — use it for almost everything. `pressSequentially()` types character-by-character with key events; reserve it for inputs with per-keystroke logic (autocomplete, input masks) where `fill` wouldn't trigger the handlers.

## `force` — the escape hatch

```ts
// Skips actionability checks (visibility, stability, receives-events, enabled):
await page.getByRole('button', { name: 'Submit' }).click({ force: true });
```

`force: true` tells Playwright "click regardless." It's occasionally legitimate (an element the checks misjudge), but usually it's papering over a real problem — an overlay intercepting the click, an element that's actually disabled, an animation still running. If you reach for `force`, first ask whether the test just caught a genuine UX bug. Fixing the wait/locator is almost always better than forcing.

## Interview Q&A

**Q: What exactly does Playwright wait for before a click?**
A: It runs actionability checks and retries until they pass: the element is attached to the DOM, visible, stable (bounding box settled across animation frames), the actual hit target at the click point (not covered by an overlay), and enabled. Only then does it dispatch the click. If they never pass within the action timeout, it throws explaining which check failed.

**Q: Why is `waitForTimeout` an anti-pattern?**
A: It couples the test to a guessed duration. Too short → flaky on a slow run; too long → the suite crawls. It synchronizes on *time* instead of *state*. Auto-waiting actions plus auto-retrying assertions synchronize on the actual condition, so they're both faster and more reliable. `waitForTimeout` is fine only for debugging or deliberately observing the absence of a change — never as a sync mechanism.

**Q: A click "works manually but fails in the test" — how do you reason about it?**
A: The error names the failed actionability check. "Intercepts pointer events" → an overlay/cookie banner/modal is on top; dismiss it or target the right element. "Not visible" → it's off-screen or not rendered yet (often the assertion/locator is wrong, or you need to wait for a real condition). "Not stable" → it's mid-animation. I'd open the trace ([trace viewer](./12-debugging-and-trace-viewer.md)) to see the DOM at the action moment. Forcing the click would hide whichever of these is the real issue.

**Q: When is `force: true` justified?**
A: Rarely — when the actionability model is demonstrably wrong for a quirky element, and you've confirmed there's no genuine UX problem. Most uses of `force` are hiding an overlay, a disabled state, or an animation, i.e. masking a real bug. Default answer: don't force; fix the precondition.

**Q: `fill` vs `pressSequentially`?**
A: `fill` sets the value atomically and fires input/change events — fast and correct for nearly all inputs. `pressSequentially` emits real per-character keystrokes; use it only when the field reacts to each keystroke (autocomplete, masked inputs) and `fill` wouldn't trigger that logic.

## Gotchas & Anti-patterns

- **`page.waitForTimeout(ms)` to "let things settle"** — the canonical flaky pattern. Wait on a condition or rely on auto-waiting/auto-retrying assertions.
- **`force: true` to get past a failing click** — usually masks an overlay, disabled control, or animation (often a real bug). Investigate the actionability error instead.
- **`expect(await locator.isVisible()).toBe(true)` then acting** — `isVisible()` is a one-shot, non-waiting boolean; it races. Use `await expect(locator).toBeVisible()` ([assertions](./05-web-first-assertions.md)).
- **Simulating typing with low-level key events when `fill` would do** — slower and skips correct event semantics.
- **Waiting for `networkidle` as a default sync** — discouraged and flaky on apps with long-polling/analytics; wait for the specific UI state or response instead.
- **Assuming auto-wait covers app-level async** — Playwright waits for actionability, not for *your* spinner's business logic. Assert on the resulting UI state to bridge that gap.

## Further Reading

- [Auto-waiting & actionability](https://playwright.dev/docs/actionability)
- [Actions](https://playwright.dev/docs/input)
- [Navigations & waiting](https://playwright.dev/docs/navigations)
