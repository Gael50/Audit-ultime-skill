# Playwright Recipes — Audit Ultime

Concrete browser scenarios. Adapt to the project, do not run blindly. Every recipe is a *building block* — combine them into the scenario battery (SKILL.md step 4).

Tool priority for execution:
1. `mcp__plugin_playwright_playwright__*` (preferred — full Playwright)
2. `mcp__browsermcp__*`
3. Local `npx playwright` (write a `audit-test.spec.js` and run with `--reporter=list --headed`)

When using local Playwright, prefer `--headed` so the user can watch. Use `--reporter=html` for the final HTML trace.

---

## Recipe 1 — Cold Load + Console Capture

```js
// Goal: confirm the app loads and capture initial errors
await page.goto(BASE_URL, { waitUntil: 'networkidle' });
const consoleErrors = [];
page.on('console', msg => { if (msg.type() === 'error') consoleErrors.push(msg.text()); });
const failedRequests = [];
page.on('response', r => { if (r.status() >= 400) failedRequests.push(`${r.status()} ${r.url()}`); });
await page.screenshot({ path: 'audit-artifacts/01-cold-load.png', fullPage: true });
// Report: page title, console errors, failed requests, total DOM nodes
const domSize = await page.evaluate(() => document.querySelectorAll('*').length);
```

**MCP equivalent:** `browser_navigate(url)` → `browser_console_messages()` → `browser_take_screenshot()` → `browser_evaluate('document.querySelectorAll("*").length')`.

---

## Recipe 2 — Auth Flow (if login page detected)

```js
// Detect login page by URL pattern or input[type=password]
if (await page.locator('input[type="password"]').count() > 0) {
  await page.fill('input[name="username"], input[type="email"]', TEST_USER);
  await page.fill('input[type="password"]', TEST_PASS);
  await page.click('button[type="submit"], button:has-text("Connexion"), button:has-text("Login")');
  await page.waitForURL(url => !url.pathname.includes('login'), { timeout: 5000 });
}
// Confirm session
const cookies = await page.context().cookies();
const hasAuth = cookies.some(c => /auth|session|token/i.test(c.name));
```

Source for `TEST_USER`/`TEST_PASS`: CLAUDE.md, README, `.env.example`, or ask user once.

---

## Recipe 3 — Main Navigation

```js
// Click every primary nav item, verify view loads
const navItems = await page.locator('nav a, nav button, [role="tab"], .tab-item').all();
const results = [];
for (let i = 0; i < navItems.length; i++) {
  const label = await navItems[i].textContent();
  await navItems[i].click();
  await page.waitForLoadState('networkidle');
  const errors = [...consoleErrors];
  consoleErrors.length = 0;
  await page.screenshot({ path: `audit-artifacts/nav-${i}-${label.replace(/\s+/g,'_')}.png` });
  results.push({ label, errors, dom: await page.evaluate(() => document.querySelectorAll('*').length) });
}
```

**Anomaly detection:**
- Same content shown after click → click had no effect (broken nav)
- Console error appears → root cause investigation needed
- DOM size jumps massively → possible "render all on first visit" anti-pattern

---

## Recipe 4 — Active State Persistence (CRITICAL)

```js
// Hard reload while on non-default view, confirm state restored
await page.click('[data-tab="settings"]'); // or any non-default
await page.waitForTimeout(200);
const beforeReload = await page.evaluate(() => location.href + ' | active: ' + document.querySelector('.active, .is-active')?.textContent);
await page.reload({ waitUntil: 'networkidle' });
await page.waitForTimeout(1000); // generous — observers may need time
const afterReload = await page.evaluate(() => location.href + ' | active: ' + document.querySelector('.active, .is-active')?.textContent);
// Compare. If different → state lost on reload (confirmed bug class)
```

This single recipe catches the entire "tab persistence" bug family.

---

## Recipe 5 — CRUD Round-Trip

```js
// Create
await page.click('button:has-text("Ajouter"), button:has-text("Nouveau"), button:has-text("Add")');
await page.fill('input[name="name"], input[placeholder*="nom" i]', 'AUDIT-TEST-' + Date.now());
await page.click('button:has-text("Enregistrer"), button:has-text("Sauvegarder"), button[type="submit"]');
const created = await page.locator('text=AUDIT-TEST-').count();
// Reload
await page.reload({ waitUntil: 'networkidle' });
const persisted = await page.locator('text=AUDIT-TEST-').count();
// Edit
// ... interact with edit UI ...
await page.reload();
const editPersisted = /* check the edited value */;
// Delete
// ... interact with delete UI ...
await page.reload();
const deletedStillGone = (await page.locator('text=AUDIT-TEST-').count()) === 0;
```

Each step is an assertion. Each failure is a confirmed bug.

---

## Recipe 6 — Filters + Search (Detect "Visual-Only" Filters)

```js
// Capture row count without filter
const noFilterCount = await page.locator('table tbody tr, .item, [data-row]').count();
// Apply filter
await page.click('[data-filter="status:active"], button:has-text("Actif")');
await page.waitForTimeout(300);
const filteredCount = await page.locator('table tbody tr, .item, [data-row]').count();
// If filteredCount === noFilterCount AND no visual change to row content → filter is decorative (confirmed illusion)
// If filteredCount === noFilterCount BUT row content changed (e.g. dimmed) → filter is visual-only on items, not on data
```

This recipe is the canonical illusion-of-functionality detector.

---

## Recipe 7 — Settings Persistence

```js
// Open settings, change one setting, reload, confirm applied
await page.click('[data-settings], button[aria-label*="param" i], button[aria-label*="setting" i]');
const before = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
await page.click('[data-theme="dark-purple"]'); // or whatever theme switch exists
await page.waitForTimeout(200);
const afterSet = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
await page.reload();
await page.waitForTimeout(500);
const afterReload = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
// Pass: afterSet !== before AND afterReload === afterSet
// Fail (illusion): afterSet !== before BUT afterReload === before → setting saved but never re-read
```

---

## Recipe 8 — Form Validation

```js
// Submit empty
await page.click('button:has-text("Ajouter")');
await page.click('button[type="submit"]');
const emptyError = await page.locator('.error, [role="alert"], .invalid-feedback').count();
// Submit invalid
await page.fill('input[type="email"]', 'not-an-email');
await page.click('button[type="submit"]');
const invalidError = /* same check */;
// Submit valid
await page.fill('input[type="email"]', 'audit@test.com');
await page.fill('input[required]', 'value');
await page.click('button[type="submit"]');
const success = /* check for success indicator */;
```

---

## Recipe 9 — Mobile Viewport (375×812)

```js
await page.setViewportSize({ width: 375, height: 812 });
await page.reload();
// Re-run main nav recipe + 1 critical CRUD flow
```

Catches:
- Burger menu broken
- Touch targets too small
- Modal not scrollable
- Bottom nav overlapping content

---

## Recipe 10 — Empty State

```js
// Navigate to a view that should be empty (clear filters, switch to new user, etc.)
// OR: clear localStorage and reload
await page.evaluate(() => localStorage.clear());
await page.reload();
// Confirm: no crash, no infinite spinner, friendly empty state
const errors = consoleErrors.length;
const spinnerStillSpinning = await page.locator('.spinner, [role="progressbar"]').isVisible();
```

---

## Recipe 11 — Network Audit

```js
// Capture all requests during a flow
const requests = [];
page.on('request', r => requests.push({ url: r.url(), method: r.method() }));
page.on('response', r => requests.push({ url: r.url(), status: r.status() }));
// Run a flow, then analyze:
// - Duplicate requests (N requests for same URL = client-side fetch loop?)
// - Failed requests (4xx/5xx)
// - Endpoints returning empty arrays consistently (stub?)
// - Long requests (>2s = perf issue or stuck)
```

---

## Recipe 12 — Differential (Same Action, Different States)

```js
// Run the same flow under N conditions, compare results
const conditions = [
  { name: 'desktop-fresh', viewport: {width: 1280, height: 800}, clear: true },
  { name: 'desktop-cached', viewport: {width: 1280, height: 800}, clear: false },
  { name: 'mobile-fresh', viewport: {width: 375, height: 812}, clear: true },
];
const results = {};
for (const c of conditions) {
  await page.setViewportSize(c.viewport);
  if (c.clear) await page.evaluate(() => { localStorage.clear(); sessionStorage.clear(); });
  await page.reload();
  // Run flow
  // Capture observable state
  results[c.name] = { dom: ..., consoleErrors: ..., visibleNav: ... };
}
// Diff results across conditions, flag divergences
```

---

## MCP Playwright tool mapping

| Native Playwright | MCP equivalent |
|-------------------|----------------|
| `page.goto(url)` | `browser_navigate(url)` |
| `page.click(selector)` | `browser_click(element, ref)` (use `browser_snapshot` first to get refs) |
| `page.fill(selector, value)` | `browser_type(element, ref, text)` or `browser_fill_form(fields)` |
| `page.screenshot()` | `browser_take_screenshot()` |
| `page.evaluate(fn)` | `browser_evaluate(fn)` |
| `page.on('console')` | `browser_console_messages()` (poll after action) |
| `page.on('response')` | `browser_network_requests()` (poll after action) |
| `page.setViewportSize()` | `browser_resize(width, height)` |
| `page.reload()` | `browser_navigate(currentUrl)` |
| `page.waitForLoadState` | `browser_wait_for(textOrCondition)` |

Always start an MCP browser session with `browser_snapshot()` to get an accessibility tree with refs before interacting.

---

## Local Playwright fallback

If no MCP browser tool, write `audit-test.spec.js` and run:

```bash
npx playwright install chromium  # first run only
npx playwright test audit-test.spec.js --headed --reporter=list,html
```

Then point user to `playwright-report/index.html`.

If Playwright isn't installed, propose: `npm install -D @playwright/test && npx playwright install chromium`. Ask before installing.

---

## Selecting the right battery for the project

| Project type | Mandatory recipes | Skip if absent |
|--------------|-------------------|----------------|
| Static site | 1, 3, 9, 11 | 2, 5, 7, 8 |
| SPA dashboard | 1, 3, 4, 5, 6, 7, 9, 10, 11, 12 | — |
| Auth-gated app | 1, 2, 3, 4, 5, 7, 11 | 10 (unless logout flow) |
| API-only (no UI) | Use `curl`/`Bash` instead of browser; check OpenAPI conformance, status codes, error shapes | All UI recipes |
| E-commerce / forms-heavy | 1, 2, 5, 6, 8, 9, 11 | — |
| Hybrid (project at hand) | Run all 12 in priority order, time-box each to 2min | — |

---

## Time budget per recipe

- Recipes 1-2: 30s each
- Recipes 3-6: 1-2min each (depends on # of nav items / filters)
- Recipes 7-10: 1min each
- Recipes 11-12: as long as the flows take

Total full audit: 15-25min of browser time on a medium SPA. Compact mode: 5-8min (recipes 1, 3, 4, 5 only).
