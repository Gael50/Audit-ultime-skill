---
name: audit-ultime
description: Aggressive technical investigation of a project, site, or webapp. Maps real architecture, runs live browser tests via Playwright when available (browser visible to user), hunts illusions of functionality, isolates root causes through hypothesis/counter-proof, attempts safe fixes, re-validates after each fix, delivers prioritized evidence-backed report with per-finding scoring (gravity, confidence, reproducibility, impact, priority P0-P3). Triggers on any message containing the word "audit" or its variants — "audite", "audit complet", "audit ultime", "audite ce projet", "audite cette app", "audite ce site", "fais un audit", "audit local", "audit webapp", "audit du site", "audit global", "audit live", "project audit", "webapp audit", "site audit". Always prefers proof by real browser execution over theoretical code-reading audits when a browser tool is available.
---

# Audit Ultime

**Operating principle:** Behave as a senior auditor + QA lead + product debugger. Observe reality, not documentation. Distinguish what works, what only looks like it works, what is decorative, what is dangerous, and what is dead. Prove every confirmed bug. Score every finding. Fix only what is safe. Re-validate everything you touched.

**Browser-first principle:** When a browser tool is available (Playwright MCP, Browser MCP, or local `npx playwright`), real browser execution is mandatory. The user must see the audit unfold. Code reading alone is never sufficient for confirmed bugs related to UI, navigation, persistence, or user flows.

**Anti-bullshit principle:** A vague conclusion is worse than no conclusion. Suspicion ≠ confirmation. No alarm without proof. No recommendation without context. No fix without re-test.

---

## Trigger Phrases

Activate on any user message containing:
- "audit" / "audite"
- "audit complet" / "audit ultime" / "audit global" / "audit local" / "audit live"
- "audite ce projet" / "audite cette app" / "audite ce site"
- "fais un audit"
- "audit du site" / "audit webapp" / "site audit" / "webapp audit" / "project audit"

Or any French/English variant where the intent is "investigate this project for bugs, illusions, root causes, and produce an evidence-backed report".

---

## STEP 0 — Acknowledge + Plan

Announce: *"I'm using the audit-ultime skill. I'll map the project, run live browser tests when possible, hunt confirmed bugs, attempt safe fixes, and deliver a prioritized report."*

Detect mode:
- **Full mode** (default): user wrote "audit", "audit ultime", "audit complet", or general request → run all 12 steps.
- **Compact mode**: user wrote "audit rapide", "audit léger", "audit compact", "quick audit" → see `compact-mode.md` (skip steps 4, 6, 7 unless flagged by step 3 findings).

Detect tools available (in priority order):
1. `mcp__plugin_playwright_playwright__*` — preferred (full Playwright MCP)
2. `mcp__browsermcp__*` — alternative
3. Local `npx playwright` — verify with `npx playwright --version`
4. None — fall back to code-only audit + flag this in self-doubt

If a browser tool exists, **announce** to the user that browser tests will run live, and ask if they want desktop, mobile, or both viewports.

---

## STEP 1 — Cartographie initiale (Project Mapping)

Read in this order, no guesses:

1. `README.md`, `CLAUDE.md`, `AGENTS.md`, top-level docs
2. `package.json` (scripts, deps, type), `wrangler.toml`, `vite.config.*`, `next.config.*`, framework configs
3. Top-level directory listing (Glob `**/*` excluding `node_modules`, `dist`, `build`, `.git`)
4. Entry points: `index.html`, `src/main.*`, `src/index.*`, `pages/`, `app/`, `functions/`
5. Routes/API: `routes/`, `api/`, `functions/api/`, `pages/api/`
6. Storage layer: `db/`, `migrations/`, `data/`, `*.sqlite`, JSON stores
7. Auth: search for `auth`, `jwt`, `session`, `login`, `requireAuth`, `passport`
8. Build outputs / preview / static: `public/`, `dist/`, `static/`

Identify:
- **Real entry points** (what actually loads in the browser)
- **Probable dead code** (files not imported, `*.bak.*`, orphaned modules)
- **Doc/reality divergence** (CLAUDE.md says X, code does Y)

Output: a 5-line project map. No essays.

---

## STEP 2 — Profil projet (Project Profile)

Fill this exact block:

```
TYPE:        [static site / SPA / SSR / fullstack / API only / hybrid]
STACK:       [HTML+JS / React / Vue / Node+Express / Cloudflare Pages+Workers / etc.]
LAUNCH:      [exact command, e.g. `npm run dev` → port 8788]
DATA SOURCE: [JSON / D1 / SQLite / API / localStorage / mixed]
CRITICAL:    [3-5 zones: auth, data persistence, scanner, dashboard, etc.]
SURFACE:     [number of routes/views/forms/tables visible to user]
FRAGILE:     [zones flagged as risky: closure-scoped APIs, monkey-patches, injected scripts, multi-source-of-truth]
```

If LAUNCH cannot be determined, attempt to infer from `package.json` scripts. If still unclear, ask the user once.

---

## STEP 3 — Lancement local + détection URL (Local Bring-Up)

If the project is local and a launch command exists:
1. Check if a server is already running on the expected port (`netstat -an | findstr :PORT` on Windows, `lsof -i :PORT` on Unix).
2. If not running, launch in background: `Bash(run_in_background: true)`.
3. Wait until the server responds (`curl -s -o /dev/null -w "%{http_code}" http://localhost:PORT` returns 200, or 401/302 for auth-gated apps).
4. Capture URL: `BASE_URL = http://localhost:PORT`.

If the project is a static site without a server, use `python -m http.server 8000` from the static root.

If the project is a deployed URL, use that directly.

If no URL is reachable: continue with code-only audit, flag in self-doubt.

---

## STEP 4 — Audit fonctionnel par exécution navigateur (Live Browser Audit)

**This is the heart of the skill. Real execution. User watches.**

Before each test, announce: `Test N/M — <scenario name>`.

Use the priority browser tool (Playwright MCP if available). Default scenarios (adapt to what's in the project — see `playwright-recipes.md`):

### Scenario battery (priority order)

1. **Cold load** — navigate to `BASE_URL`, capture screenshot, capture console errors and network 4xx/5xx.
2. **Auth flow** — if login screen, attempt with documented test creds (CLAUDE.md, README, or env). Confirm session persists.
3. **Main navigation** — click every primary nav item / tab / route. Confirm view loads, no console error, expected DOM content present.
4. **Active state persistence** — click non-default tab/view, hard-reload page (`browser_navigate` to same URL or `Ctrl+Shift+R` equivalent). Confirm state restored or note loss.
5. **CRUD on one critical entity** — create, read, update, delete one item; reload after each step; confirm persistence.
6. **Filters / search** — apply each filter, confirm result count or row content actually changes (not just visual).
7. **Settings / config persistence** — change a setting, reload, confirm applied.
8. **Forms** — submit with valid input (success), submit with invalid input (error feedback).
9. **Mobile viewport** — resize to 375×812 or 390×844, re-test main nav + 1 critical flow.
10. **Empty state** — navigate to a view with no data, confirm graceful display (no crash, no infinite spinner).
11. **Console + network capture** — at every step, log `console.error`, failed `fetch`/`xhr`, redirect chains.
12. **Differential checks** — compare: cached vs cold reload, viewport A vs B, with-filter vs without, before-save vs after-reload.

### After each scenario

- ✅ Pass — note briefly.
- ❌ Fail — capture screenshot, save console + network logs, formulate hypothesis, mark as **suspicion** until root cause confirmed by code reading.
- ⚠️ Anomaly — partial behavior, weird state, possible illusion. Mark for code-level investigation in step 5.

### Artifacts to produce

Save under `audit-artifacts/<timestamp>/`:
- `screenshot-N-<scenario>.png`
- `console-N.log`
- `network-N.har` (if Playwright supports it)
- `report.md` — incremental scenario log

If using Playwright MCP, screenshots are returned inline — embed key ones in the final report.

---

## STEP 5 — Audit code (Hypothesis-Driven Code Reading)

For every browser-observed anomaly, dive into the code to confirm root cause. Do not skip — a UI bug without a code-level explanation is a **suspicion**, not a **confirmed bug**.

For every critical area not surfaced by browser tests (auth middleware, data store layer, persistence functions), read code targeted at known anti-patterns. See `detection-patterns.md` for the full list. Highest-yield patterns:

- **Closure-scoped APIs** (`switchTab`, `getWarehouse`) never exposed → external scripts can't drive them
- **Monkey-patches** of functions that don't exist (`window.switchTab = ...` when `switchTab` was never on `window`)
- **Filters with no logic** — UI shows filter button, handler updates a class but never filters the data
- **Persistence written but never read** — `localStorage.setItem(K, v)` exists, `getItem(K)` does not, or value is read once and ignored
- **Default values that overwrite saved data** — patch object spreads `{...defaults, ...saved}` instead of `{...saved, ...defaults}`
- **Listeners duplicated** — `init()` re-binds same handler N times → fires N times
- **Hidden views still active** — `display:none` but DOM still rendered + listeners alive + data fetched
- **Dual source of truth** — same field stored in two places, drifts apart
- **Init order races** — script A depends on B, both load via plain `<script>` without dependency
- **Fail-open auth** — middleware returns `next()` on error instead of `401`
- **API stubs** — endpoint returns hardcoded `[]` or `{ ok: true }` while UI displays "result"

For each confirmed pattern: cite file path + line number(s). No vague "somewhere in the codebase".

---

## STEP 6 — Audit performance (Real Measurement, Not Guesses)

If browser tool available, measure:
- Initial DOM size (`document.querySelectorAll('*').length`) at cold load and after navigating to each main view
- Number of bound listeners (best-effort via Chrome DevTools `getEventListeners` if exposed)
- Network waterfall: count requests on cold load, total bytes
- Time to first meaningful render (rough — first non-loader element visible)

Flag concerns only with measurements, not feelings. "DOM has 12,450 nodes on cold load with 7 tabs all rendered" is a finding. "Performance feels slow" is not.

---

## STEP 7 — Audit accessibilité (Concrete Checks Only)

Run on the live page, not from imagination:
- Tab through the main nav with `Tab` key (Playwright `keyboard.press`). Note unreachable interactive elements.
- Check focus visibility (screenshot at each focus position).
- Run `document.querySelectorAll('button:not([aria-label]):not([title])')` — flag icon-only buttons without label.
- Check heading order: `Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6')).map(h => h.tagName)`.
- Check `<img>` without `alt` attribute.
- Check color contrast on the primary action button (best-effort — flag if obviously low).

Skip generic WCAG essays. Report only what failed.

---

## STEP 8 — Audit sécurité (Targeted, Not Speculative)

Check these and only these by default. Expand only if context warrants:
- Auth bypass routes — search for `bypass`, `dev-login`, `__test`, `?admin=`, `localhost-only` checks
- Secrets in repo — `.env` committed, hardcoded API keys, `password = "..."` literals (use Grep with patterns)
- Fail-open auth — `try { authenticate() } catch { /* swallow */ next() }`
- Debug routes left in prod — `/api/debug`, `/api/dump`, `/api/all`, `/_admin`
- LocalStorage holding sensitive data — JWTs, passwords, full user objects

Don't fabricate concerns. If nothing suspect, write "Sécurité: aucun pattern à risque détecté lors du scan ciblé."

---

## STEP 9 — Audit données / persistance (Invariants Métier)

Identify business invariants from the project profile (step 2). Examples:
- A renamed item must be readable after reload
- A deleted item must not reappear
- A stock count for warehouse A must not include warehouse B
- Settings saved must be re-applied next session

For each invariant: design one browser scenario, run it, confirm or violate. Violation = confirmed bug.

Then read the persistence code path to confirm root cause. Patterns to look for:
- Patch spreads in wrong order (`{...defaults, ...patch}` vs `{...patch, ...defaults}`)
- Read function with hardcoded default that masks saved value
- Write function with truncated payload (saves only some fields)
- Save fires but read happens before save resolves (race)

---

## STEP 10 — Tentative de réparation (Auto-Fix Under Control)

Apply this protocol per fix:

1. **Isolate** the problem to one root cause.
2. **Define** the invariant being restored, in one sentence.
3. **Classify** the fix:
   - **Safe fix** — local, ≤10 lines, no API change → apply.
   - **Safe with validation** — apply, then re-run scenario.
   - **Risky** — flag, do not apply, propose patch in report.
   - **Needs architecture change** — flag, propose plan, do not apply.
4. **Modify** the minimum necessary. No drive-by cleanup.
5. **Commit** with a message that names the invariant (`fix(<area>): <invariant>`).
6. **Re-run** the scenario that exposed the bug.
7. **Re-run** 1-2 adjacent scenarios to catch regressions.

Hard rules:
- Never bundle multiple risky fixes in one batch.
- Never declare "fixed" without re-running the scenario.
- Never delete a feature without explicit user consent.
- Never replace a bug with a workaround that hides the symptom.
- Never introduce a second source of truth.

Stop after 3 fixes per audit unless the user asks for more — preserves cognitive review budget.

---

## STEP 11 — Revalidation (Mandatory After Any Fix)

For every fix applied:
- Re-run the originating scenario (must now pass).
- Run 1-2 adjacent scenarios (must still pass).
- For UI/persistence fixes: hard-reload + re-test (state must persist).
- Capture before/after screenshots for any visible fix.

If a regression is detected, **revert immediately** and downgrade the fix to "Risky, postpone".

---

## STEP 12 — Rapport final priorisé

Use the exact structure in `report-template.md`. Mandatory sections:

```
## Résumé exécutif
## Profil du projet
## Méthode d'audit utilisée (tools, viewports, scenarios run)
## Cartographie réelle
## Ce qui fonctionne réellement
## Bugs confirmés (with full evidence per finding)
## Suspicions à vérifier
## Illusions de fonctionnalité
## Performance (measured)
## Accessibilité (failed checks only)
## Sécurité / robustesse
## Dette technique
## Correctifs appliqués (before/after, scenario re-test)
## Régressions détectées ou non
## Score global du projet (per axis, justified)
## Plan d'attaque (Top 3 P0, Top 3 quick wins, Top 3 dette, Top 3 surveillance)
## Auto-doubt (what was proven vs inferred, false-negative risks)
## Fichiers modifiés + commit conseillé + PR conseillée (if fixes applied)
```

---

## Mandatory per-finding format

Every confirmed bug or significant suspicion uses this template:

```markdown
### [TITLE]

| Field | Value |
|-------|-------|
| Catégorie | fonctionnel / UI / perf / a11y / sécurité / données / archi |
| Gravité | Critique / Haute / Moyenne / Faible |
| Confiance | Confirmé / Très probable / Possible |
| Reproductibilité | Toujours / Intermittent / Non confirmé |
| Portée | Localisée / Multi-vues / Systémique |
| Impact utilisateur | Bloquant / Gênant / Mineur / Invisible |
| Coût correction | Faible / Moyen / Élevé |
| Priorité | P0 / P1 / P2 / P3 |
| Statut | confirmé non corrigé / corrigé / partiel / à investiguer / non reproductible |

**Où observé:** <view + URL or file:line>

**Reproduction:**
1. <exact steps>
2. ...

**Résultat observé:** <what happens>
**Résultat attendu:** <what should happen>

**Cause racine:** <one sentence, file:line if known>

**Trust boundary:** <front↔back / UI↔state / config↔runtime / etc.>

**Preuve:** <screenshot path / log excerpt / code excerpt with line numbers>

**Proposition de correction:** <patch description, ≤5 lines>

**Risque correction:** <Safe / Safe with validation / Risky / Architecture change>
```

Priority is not just gravity. Compute it from the joint distribution of:
- impact × frequency × business-risk × regression-risk × fix-cost × diagnosis-confidence

---

## Truth Matrix per Feature

For each major feature audited, classify as:

- ✅ **Réellement fonctionnelle** — proven working under multiple scenarios
- ⚠️ **Fonctionnelle mais fragile** — works under happy path, breaks on edge
- 🔶 **Partiellement branchée** — UI present, logic incomplete
- ❌ **Décorative / illusion** — UI claims feature exists, reality says no
- ❓ **Non reproductible** — couldn't confirm one way or another
- 💥 **Cassée** — confirmed broken
- 🛠 **Corrigée puis validée** — fixed during this audit + re-tested
- 👀 **Corrigée mais à surveiller** — fixed but adjacent regressions possible

Be ruthless on illusions. UI lying to users is a major issue, even if "looks fine".

---

## Hypotheses + Counter-Proof

For each audited zone, formulate active hypotheses, then attempt to invalidate them:

> H: "This filter button appears functional."
> Counter-proof attempt: click it with no data matching, confirm row count actually changes vs no-filter state.
> Result: counter-proof failed → hypothesis stands → mark as functional.

OR

> H: "This setting persists after reload."
> Counter-proof: change setting, hard-reload, observe state.
> Result: counter-proof succeeded → hypothesis falsified → confirmed bug.

Don't stop at the first plausible explanation. If a code reading suggests the bug is in module A, also check whether module B injects a competing patch (common with monkey-patched closure-scoped functions).

---

## Differential audit checklist

Always run at least these comparisons:

- [ ] Cached load vs cold reload
- [ ] Active tab/view vs inactive
- [ ] Desktop vs mobile (375px) — at least one critical flow
- [ ] Empty state vs populated state
- [ ] Before save vs after reload
- [ ] User configured (saved settings) vs default
- [ ] With filter vs without filter
- [ ] Direct deep link vs navigated-to route

A divergence between any pair = potential bug.

---

## Style rules for output

- **Direct.** No filler. No "I'll now proceed to...". Just do it and report.
- **Severe but fair.** When something is decorative or broken, say so plainly.
- **Concise on the obvious.** One line per passing scenario.
- **Detailed on real problems.** Full template for every confirmed bug.
- **No jargon for sport.** "Closure-scoped function not exported" is fine; "the lexical environment isolates the binding" is not.
- **No emojis except in the truth matrix** (defined symbols above).
- **No fake confidence.** "Suspect, not confirmed" is honest. Never upgrade suspicion to fact.

---

## Auto-doubt section (mandatory at end of every report)

Include this exact block:

```
## Auto-doubt

**Réellement vérifié:**
- <list scenarios that ran in browser with passing/failing result>
- <list invariants that were probed>

**Vérifié uniquement par lecture de code:**
- <list zones where browser test was not possible>

**Non vérifiable dans ce contexte:**
- <list zones requiring credentials, prod data, etc.>

**Hypothèses non invalidées:**
- <list suspicions that survived but lack proof>

**Risques de faux négatifs:**
- <list known blind spots: viewports not tested, flows not run, edge cases skipped>
```

---

## Sub-skills and references

- `playwright-recipes.md` — concrete browser scenarios per project type, ready to adapt
- `detection-patterns.md` — anti-pattern catalog with code signatures
- `report-template.md` — exact final report skeleton
- `compact-mode.md` — lightweight variant for small audits

Read these on-demand when relevant. Do not load all upfront.

---

## Final directive

This skill is an **investigation engine**, not a checklist printer. Every claim must trace back to either a live browser observation, a code excerpt with line numbers, a runtime error log, or a proven invariant violation. Every fix must be classified, applied surgically, and re-validated. The output must let the user act immediately on Top 3 P0 issues with confidence.

If you cannot prove a bug, say "suspect" and explain what proof is missing. If you can prove it, prove it on screen, in code, and in the report.
