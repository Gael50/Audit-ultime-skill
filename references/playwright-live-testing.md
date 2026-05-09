# Playwright Live Testing — Audit Ultime

Comment tester l'app en navigateur visible et utiliser ces tests à la fois pour **découvrir** des bugs (step 4) et **valider** les correctifs (step 11).

Principe : visibilité maximale pour l'utilisateur. Les tests ne sont pas opaques. L'utilisateur voit l'audit se dérouler.

---

## Tool priority

1. `mcp__plugin_playwright_playwright__*` — préféré (Playwright MCP complet)
2. `mcp__browsermcp__*` — alternative
3. `npx playwright` local — écrire un `audit-test.spec.js` + lancer avec `--reporter=list,html --headed`
4. Aucun → audit code-only ; flagger dans auto-doubt + dans le rapport

Quand local Playwright : préférer `--headed` pour que l'utilisateur observe. `--reporter=html` pour la trace HTML finale.

Si Playwright n'est pas installé, **demander avant** d'installer : `npm install -D @playwright/test && npx playwright install chromium`.

---

## Visibilité utilisateur

Avant chaque test, annoncer :
```
Test N/M — <nom du scénario>
```

Pendant l'exécution : reporter live les succès / échecs / anomalies. Ne pas batcher 12 tests puis livrer un seul résultat — c'est opaque.

Distinguer dans la sortie finale :
- Tests **exécutés** réellement en direct (preuve forte)
- Tests **inférés** par lecture de code seulement (preuve faible)
- Tests **non exécutables** dans ce contexte (manque de creds, env, données)

---

## Battery par défaut

12 recettes. Toutes ne s'appliquent pas à tous les projets. Voir tableau "Sélection par type" en fin de fichier.

### Recipe 1 — Cold load + console capture

Confirmer que l'app charge ; capturer les erreurs initiales.

```js
await page.goto(BASE_URL, { waitUntil: 'networkidle' });
const consoleErrors = [];
page.on('console', m => { if (m.type() === 'error') consoleErrors.push(m.text()); });
const failedRequests = [];
page.on('response', r => { if (r.status() >= 400) failedRequests.push(`${r.status()} ${r.url()}`); });
await page.screenshot({ path: 'audit-artifacts/01-cold-load.png', fullPage: true });
const domSize = await page.evaluate(() => document.querySelectorAll('*').length);
```

**MCP** : `browser_navigate(url)` → `browser_console_messages()` → `browser_take_screenshot()` → `browser_evaluate('document.querySelectorAll("*").length')`.

### Recipe 2 — Auth flow (si login détecté)

```js
if (await page.locator('input[type="password"]').count() > 0) {
  await page.fill('input[name="username"], input[type="email"]', TEST_USER);
  await page.fill('input[type="password"]', TEST_PASS);
  await page.click('button[type="submit"], button:has-text("Connexion"), button:has-text("Login")');
  await page.waitForURL(url => !url.pathname.includes('login'), { timeout: 5000 });
}
const cookies = await page.context().cookies();
const hasAuth = cookies.some(c => /auth|session|token/i.test(c.name));
```

Source `TEST_USER`/`TEST_PASS` : CLAUDE.md, README, `.env.example`. Sinon, demander une fois.

### Recipe 3 — Main navigation

```js
const navItems = await page.locator('nav a, nav button, [role="tab"], .tab-item').all();
for (let i = 0; i < navItems.length; i++) {
  const label = await navItems[i].textContent();
  await navItems[i].click();
  await page.waitForLoadState('networkidle');
  await page.screenshot({ path: `audit-artifacts/nav-${i}.png` });
  // capture errors per click
}
```

**Anomalies typiques** : même contenu après clic = nav cassée ; console error = investigation code ; DOM jump massif = anti-pattern "render all on first visit".

### Recipe 4 — Active state persistence (CRITIQUE)

```js
await page.click('[data-tab="settings"]'); // non-default
await page.waitForTimeout(200);
const before = await page.evaluate(() => location.href + ' | active: ' + document.querySelector('.active, .is-active')?.textContent);
await page.reload({ waitUntil: 'networkidle' });
await page.waitForTimeout(1000);
const after = await page.evaluate(() => location.href + ' | active: ' + document.querySelector('.active, .is-active')?.textContent);
// Différence → état perdu au reload (bug confirmé)
```

Cette recette à elle seule capture toute la famille de bugs "tab perdu au refresh".

### Recipe 5 — CRUD round-trip

```js
await page.click('button:has-text("Ajouter"), button:has-text("Nouveau")');
await page.fill('input[name="name"], input[placeholder*="nom" i]', 'AUDIT-' + Date.now());
await page.click('button:has-text("Enregistrer"), button[type="submit"]');
const created = await page.locator('text=AUDIT-').count();
await page.reload({ waitUntil: 'networkidle' });
const persisted = await page.locator('text=AUDIT-').count();
// edit + reload + verify
// delete + reload + verify
```

Chaque étape = une assertion. Chaque échec = bug confirmé.

### Recipe 6 — Filtres (détection des filtres décoratifs)

```js
const noFilter = await page.locator('table tbody tr, .item, [data-row]').count();
await page.click('[data-filter="status:active"], button:has-text("Actif")');
await page.waitForTimeout(300);
const filtered = await page.locator('table tbody tr, .item, [data-row]').count();
// Si filtered === noFilter ET aucun changement de contenu visible → filtre décoratif (illusion confirmée)
```

Détecteur canonique des illusions de fonctionnalité.

### Recipe 7 — Settings persistence

```js
await page.click('[data-settings], button[aria-label*="param" i]');
const before = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
await page.click('[data-theme="dark-purple"]');
await page.waitForTimeout(200);
const afterSet = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
await page.reload();
await page.waitForTimeout(500);
const afterReload = await page.evaluate(() => getComputedStyle(document.body).getPropertyValue('--accent-primary'));
// Pass: afterSet !== before AND afterReload === afterSet
// Fail (illusion): afterSet !== before BUT afterReload === before
```

### Recipe 8 — Form validation

Submit empty / invalid / valid. Vérifier feedback d'erreur (`.error`, `[role="alert"]`, `.invalid-feedback`) + indicateur de succès.

### Recipe 9 — Mobile viewport (375×812)

```js
await page.setViewportSize({ width: 375, height: 812 });
await page.reload();
// Re-run nav + 1 critical CRUD
```

Détecte : burger cassé, touch targets trop petits, modal non scrollable, bottom nav qui chevauche.

### Recipe 10 — Empty state

```js
await page.evaluate(() => localStorage.clear());
await page.reload();
const errors = consoleErrors.length;
const spinner = await page.locator('.spinner, [role="progressbar"]').isVisible();
// Pass: pas de crash, pas de spinner infini, empty state propre
```

### Recipe 11 — Network audit

Capturer toutes les requêtes pendant un flux. Analyser :
- Requêtes dupliquées (boucle fetch ?)
- 4xx/5xx
- Endpoints retournant `[]` systématiquement (stub ?)
- Requêtes longues (>2s)

### Recipe 12 — Différentiel (même action, états différents)

Exécuter le même flux sous N conditions, comparer.

```js
const conditions = [
  { name: 'desktop-fresh', viewport: {width:1280,height:800}, clear: true },
  { name: 'desktop-cached', viewport: {width:1280,height:800}, clear: false },
  { name: 'mobile-fresh',  viewport: {width:375,height:812}, clear: true },
];
// Pour chaque, run le flux, capturer DOM/console/visible state
// Diff entre conditions = divergences = bugs potentiels
```

---

## MCP Playwright tool mapping

| Native Playwright | MCP equivalent |
|---|---|
| `page.goto(url)` | `browser_navigate(url)` |
| `page.click(selector)` | `browser_snapshot()` puis `browser_click(element, ref)` |
| `page.fill(selector, value)` | `browser_type(element, ref, text)` ou `browser_fill_form(fields)` |
| `page.screenshot()` | `browser_take_screenshot()` |
| `page.evaluate(fn)` | `browser_evaluate(fn)` |
| `page.on('console')` | `browser_console_messages()` (poll après action) |
| `page.on('response')` | `browser_network_requests()` (poll après action) |
| `page.setViewportSize()` | `browser_resize(width, height)` |
| `page.reload()` | `browser_navigate(currentUrl)` |
| `page.waitForLoadState` | `browser_wait_for(textOrCondition)` |

Toujours démarrer une session MCP par `browser_snapshot()` pour récupérer l'arbre d'accessibilité avec refs.

---

## Sélection par type de projet

| Projet | Recettes obligatoires | À skipper |
|--------|----------------------|-----------|
| Site statique | 1, 3, 9, 11 | 2, 5, 7, 8 |
| SPA dashboard | 1, 3, 4, 5, 6, 7, 9, 10, 11, 12 | — |
| App auth-gated | 1, 2, 3, 4, 5, 7, 11 | 10 (sauf flow logout) |
| API only | curl/Bash, conformance OpenAPI, status, error shapes | toutes les recettes UI |
| E-commerce / forms | 1, 2, 5, 6, 8, 9, 11 | — |
| Hybride | toutes en priorité, time-box 2 min chacune | — |

---

## Budget temps

- Recettes 1-2 : 30 s chacune
- Recettes 3-6 : 1-2 min chacune (selon nb d'items / filtres)
- Recettes 7-10 : 1 min chacune
- Recettes 11-12 : durée des flux

Total full audit : 15-25 min navigateur sur SPA moyenne. Compact : 5-8 min (recettes 1, 3, 4, 5).

---

## Artefacts à produire

Sous `audit-artifacts/<timestamp>/` :
- `screenshot-N-<scenario>.png`
- `console-N.log`
- `network-N.har` (si Playwright natif)
- `report.md` — log incrémental

Rapport HTML Playwright si dispo : pointer le user vers `playwright-report/index.html`.

Trace viewer : capture la timeline de chaque test. À privilégier pour les bugs intermittents.

---

## Revalidation post-fix (step 11)

Pour chaque bug UI / navigation / persistance / interaction corrigé :

1. Rejouer la recette qui exposait le bug
2. Capturer screenshot avant/après si visible
3. Vérifier la console (aucune nouvelle erreur)
4. Vérifier la persistance après reload
5. Vérifier 1-3 flux adjacents

Sans browser dispo → confiance du fix limitée à **moyenne** (cf. `repair-rules.md`).

---

## Tests de non-régression à conserver

Après audit, proposer à l'utilisateur d'intégrer en CI :
- La recette 4 (active state persistence) — couvre une famille entière
- La recette 7 (settings persistence) — couvre une autre famille entière
- 1-2 recettes spécifiques aux bugs trouvés et corrigés pendant cet audit

Ces tests **persistent au-delà de l'audit** et empêchent les régressions futures sur les zones sensibles.

---

## Mode local first

Si projet local :
- Détecter comment lancer
- Réutiliser un serveur déjà actif si possible (vérifier le port)
- Identifier l'URL locale exacte
- Attendre que l'app soit prête
- Puis tester

Éviter :
- Supposer un port au hasard
- Lancer plusieurs serveurs concurrents
- Conclure sans avoir réellement testé la page chargée

---

## Multi-vue / multi-viewport obligatoire

Si pertinent, exécuter au minimum :
- 1 test desktop
- 1 test mobile (375px)
- 1 test refresh / persistance
- 1 test changement de vue / onglet
- 1 test retour arrière simple

Détecte : bugs responsive, états non persistés, vues mal montées, éléments cachés mais cassés, divergences mobile / desktop.
