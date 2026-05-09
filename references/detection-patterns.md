# Detection Patterns — Audit Ultime

Concrete bug signatures to hunt with Grep + Read. Each pattern: name, signature, why it's a bug, how to confirm, severity heuristic.

Use this as a checklist when running step 5 (code audit). Don't run them all blindly — prioritize patterns relevant to anomalies surfaced by the browser audit.

---

## P1 — Closure-scoped function never exported

**Signature:**
```js
const switchTab = (tab) => { ... }; // inside an IIFE / module closure
// And nowhere: window.switchTab = switchTab;
// Then elsewhere: window.switchTab?.(tab); // call site assumes global
```

**Grep recipe:**
```
grep -n "window\.\w\+\s*=" <main-script>   # find all window assignments
grep -n "const \w\+\s*=\s*(.*=>" <main-script>  # candidates for unexported
```

**Why bug:** External code (other scripts, console, automation) can't drive the function. UI works only via internal click handlers; any "window.X" call is a no-op.

**Severity:** Often **Critique** if it gates user-visible features (tab switching, persistence restore).

**Fix:** Either expose with `window.X = X;` or rebuild the integration through DOM events (`.click()`).

---

## P2 — Monkey-patch on a non-existent target

**Signature:**
```js
window.switchTab = function(tab) { /* override */ };
// But switchTab was never on window in the first place
```

**Confirmation:** Search for the original definition. If it's inside a closure with no `window.` assignment, the monkey-patch overrides nothing.

**Severity:** **Haute** — the patch silently no-ops, may give false sense of security.

**Fix:** Remove the dead patch or fix the original to be exportable.

---

## P3 — Persistence written but never read

**Signature:**
```js
localStorage.setItem('myKey', value); // appears
// Search for getItem('myKey') → no result, or only in dead code
```

**Grep recipe:**
```
grep -rn "setItem" --include="*.js" --include="*.html" | awk -F"'" '{print $2}' | sort -u > /tmp/keys-written
grep -rn "getItem" --include="*.js" --include="*.html" | awk -F"'" '{print $2}' | sort -u > /tmp/keys-read
diff /tmp/keys-written /tmp/keys-read
```

**Why bug:** User settings appear to save but never apply. Classic illusion of functionality.

**Severity:** **Haute** — user trust violation.

**Fix:** Add `getItem` + apply on init; or remove the misleading `setItem`.

---

## P4 — Default values overwriting saved data

**Signature:**
```js
const config = { ...defaults, ...savedFromStorage }; // CORRECT
// vs
const config = { ...savedFromStorage, ...defaults }; // BUG — defaults always win
```

**Grep recipe:**
```
grep -n "{\s*\.\.\.\w\+,\s*\.\.\.\w\+\s*}" <files>
```

**Confirmation:** Read which object is the user's saved state and which is the default. If defaults come last in the spread, they overwrite saved values.

**Severity:** **Critique** if the user observes "settings reset themselves".

**Fix:** Reorder spreads.

---

## P5 — Fail-open authentication

**Signature:**
```js
function requireAuth(req, res, next) {
  try {
    const user = decode(req.cookies.token);
    req.user = user;
    next();
  } catch {
    next();  // BUG — error swallowed, request continues unauthenticated
  }
}
```

**Severity:** **Critique** if any auth-gated route is reachable without a valid token.

**Fix:** `catch { return res.status(401).json(...); }`.

---

## P6 — Filter that doesn't filter

**Signature:**
```js
filterBtn.addEventListener('click', () => {
  filterBtn.classList.toggle('active'); // visual only
  // ... no filter logic on the data set
});
```

Or:
```js
const filtered = items.filter(i => true); // always true → no filtering
```

**Confirmation:** Inspect the filter handler. Compare row count or row content with vs without filter via browser test.

**Severity:** **Haute** (illusion of functionality).

**Fix:** Implement actual filter predicate; ensure data list is updated.

---

## P7 — Listeners duplicated on init

**Signature:**
```js
function init() {
  document.getElementById('btn').addEventListener('click', handler);
  // No removeEventListener; init() called multiple times (e.g. on warehouse switch)
}
```

**Confirmation:** Grep for `init()` call sites. If called >1× without cleanup, listeners stack.

**Manifestation:** Click triggers handler N times. User reports "it triggers twice", "it adds the row 3 times", etc.

**Severity:** **Haute** if it causes data corruption (duplicate creates, double API calls).

**Fix:** Use `removeEventListener` before re-adding, or guard `init()` with a `__inited` flag, or delegate to a single document-level listener.

---

## P8 — Hidden views still active

**Signature:**
```html
<div id="view-2" style="display:none">
  <table id="t2"><tbody></tbody></table>
</div>
<script>
function renderAll() {
  renderView1(); renderView2(); renderView3(); // all rendered, even hidden
}
</script>
```

**Confirmation:** Read the equivalent of `renderAll`. Count render calls. Check if non-active views are still populated.

**Severity:** **Moyenne** — perf hit, possible correctness issue if hidden views have side effects.

**Fix:** Render lazily (only active view), as in T6 of the previous plan.

---

## P9 — Init order race

**Signature:**
```html
<script src="/lib/storage.js"></script>
<script src="/main.js"></script>
<!-- main.js calls window.Storage.init() at top level -->
<!-- storage.js runs synchronously above, OK -->

<!-- BUT: -->
<script defer src="/lib/storage.js"></script>
<script src="/main.js"></script>
<!-- main.js may run before storage.js with defer/async on previous -->
```

**Confirmation:** Read script tag order + attributes (`defer`, `async`, `module`). Any module-level access to a global defined by an async/deferred sibling = race.

**Severity:** **Haute** — intermittent failures, hard to reproduce.

**Fix:** Either remove `defer/async`, or wrap dependent code in `window.addEventListener('load', ...)`.

---

## P10 — Dual source of truth

**Signature:** Same business field stored in two places, drifts apart.

```js
// In one module:
state.warehouse = 'La Baule';
// In another:
localStorage.setItem('ordibarre.warehouse', 'Saint-Nazaire');
// UI reads from state in one view, from localStorage in another
```

**Confirmation:** Grep the field name across the codebase. >1 owner = suspect.

**Severity:** **Haute** — when the two diverge, user sees one warehouse but data filters by another.

**Fix:** Pick one source of truth, route reads through one accessor.

---

## P11 — API stub returning fake success

**Signature:**
```js
app.get('/api/orders', (req, res) => {
  res.json({ ok: true, items: [] }); // hardcoded empty
});
```

**Grep recipe:**
```
grep -n "res\.json\(\s*{\s*\(ok:\s*true\|items:\s*\[\]\)" <api-files>
```

**Confirmation:** Run the endpoint with real data context, see if it ever returns non-trivial content.

**Severity:** **Critique** if UI claims to display orders but list is always empty.

**Fix:** Implement actual data fetch.

---

## P12 — Default value silently masks failure

**Signature:**
```js
function getStock(item) {
  return parseInt(item.stock, 10) || 0;  // any parse failure → 0
}
```

**Why bug:** A real `0` and a parse-failure `0` are indistinguishable. UI shows stock = 0 when data is actually corrupted.

**Severity:** **Moyenne** — data quality smoke screen.

**Fix:** Distinguish `undefined`/`NaN` from real `0`. Throw or log on parse failure.

---

## P13 — Patch operation overwrites unrelated fields

**Signature:**
```js
function updateItem(id, patch) {
  const item = items.find(...);
  return { ...item, ...patch, dateModification: Date.now() };
}
// Caller passes { name: 'New Name' } expecting other fields preserved.
// BUG variant: function defines defaults that overwrite:
//   return { id, name: '', description: '', ...patch }
//   ↑ if name isn't in patch, it gets reset to ''
```

**Severity:** **Critique** — silent data loss on edit.

**Fix:** Spread item last, patch first; never spread defaults after item.

---

## P14 — Forgotten debug routes

**Signature:**
```
GET /api/debug/dump
GET /api/_admin/all
GET /api/?test=1
```

**Grep recipe:**
```
grep -rn "router\.\(get\|post\)\s*(\s*['\"]\/\(debug\|_\|test\|admin\)" <api>
```

**Severity:** **Critique** if endpoints expose data without auth.

**Fix:** Remove or gate behind explicit dev-only check (`if (process.env.NODE_ENV !== 'development') return next('route')`).

---

## P15 — `&&` short-circuit hiding bugs

**Signature:**
```js
data && setData(data); // OK
// vs
0 && doSomething(); // never executes — bug if 0 is a valid value
items.length && render(items); // doesn't render empty state
```

**Severity:** **Faible** unless the missing branch causes user-visible issues.

**Fix:** Use explicit `if` with proper else branches when both states matter.

---

## P16 — useState/setState race in handlers

**Signature (React):**
```jsx
const [count, setCount] = useState(0);
const handler = () => {
  setCount(count + 1);
  setCount(count + 1); // both based on stale `count`
};
```

**Severity:** **Moyenne** — observable as "click increments by 1 instead of 2".

**Fix:** `setCount(c => c + 1)` (functional updater).

---

## P17 — Async without await + missing error path

**Signature:**
```js
async function save(data) {
  try {
    await api.post('/save', data);
  } catch (e) {
    // BUG: silent — no toast, no log, no retry
  }
}
```

**Severity:** **Haute** if the function is the save path.

**Fix:** Surface the error to the user; log to console.error at minimum.

---

## P18 — Selector fragile / coupled to copy

**Signature:**
```js
document.querySelector('button:has-text("Connexion")');
// works in French, breaks if UI translated
```

**Severity:** **Faible** unless UI has multi-language support.

**Fix:** Use stable selectors (`data-action="login"`, `id`, `aria-label`).

---

## P19 — File grown beyond review threshold

**Signature:** Any single source file over ~3000 lines.

**Detection:**
```bash
find . -name "*.js" -o -name "*.ts" -o -name "*.html" | xargs wc -l | sort -rn | head -20
```

**Why dette:** Cognitive load too high; cross-file impact analysis fails.

**Severity:** **Moyenne** dette technique. Not a bug per se, but a forcing function for future bugs.

**Fix:** Plan a split per responsibility.

---

## P20 — Duplicate logic across modules

**Signature:** Same parser/formatter/validator implemented in N places with subtle drift.

**Detection:** Grep for distinctive function names or unique strings; expect 1 hit, find 3.

**Severity:** **Moyenne** dette — when one is fixed, others remain buggy.

**Fix:** Consolidate to one shared module.

---

## Patterns specific to webapps with backend

### P21 — Front/back schema drift

**Signature:** Frontend expects `item.name`, backend returns `item.title`. UI shows `undefined`.

**Detection:** Diff API response shape (capture via browser network) against frontend usage.

### P22 — Pagination stub

**Signature:** API ignores `?page=N` and always returns full list. Frontend builds pagination UI based on a fake count.

### P23 — Cache header missing on dynamic endpoint

**Signature:** `/api/data` served with `Cache-Control: max-age=86400`. Stale data shown.

---

## Patterns specific to localStorage-heavy apps

### P24 — JSON.parse without try/catch on user data

**Signature:**
```js
const config = JSON.parse(localStorage.getItem('config'));
// Throws if value is empty string or corrupted, breaks page load
```

**Severity:** **Haute** — single bad value breaks whole app.

**Fix:** Wrap in try/catch with fallback default.

### P25 — Per-user key without user binding

**Signature:**
```js
localStorage.getItem('settings'); // global key
// User A sees User B's settings if same browser
```

**Severity:** **Haute** privacy violation.

**Fix:** Namespace key per user: `settings.${username}`.

---

## How to triage during audit

When step 5 finds a pattern hit, record:
- Pattern ID (P1–P25)
- File:line evidence
- Live observable manifestation (from step 4 browser test) — if any
- Severity per heuristic above, adjusted by impact

Combine matching patterns into one finding when they share a single root cause.

If no pattern matches but a bug is observed, write a fresh pattern entry inline in the report and flag for adding to this catalog.
