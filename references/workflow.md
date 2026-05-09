# Workflow détaillé — Audit Ultime

12 étapes opérationnelles. Chaque étape a une sortie claire. Ne pas sauter d'étape sans justification écrite. Les étapes 4 (live browser) et 11 (revalidation) sont obligatoires si un outil navigateur est disponible.

---

## STEP 0 — Acknowledge + Plan

Annonce : *"J'utilise la skill audit-ultime. Je cartographie le projet, exécute des tests navigateur live si possible, isole les bugs prouvés, tente des correctifs sûrs, livre un rapport priorisé."*

Détecter les outils dans cet ordre :
1. `mcp__plugin_playwright_playwright__*` (préféré)
2. `mcp__browsermcp__*`
3. `npx playwright` local (`npx playwright --version` pour confirmer)
4. Aucun → audit code-only + flag dans auto-doubt

Détecter le mode :
- **Full** (défaut) : "audit", "audit ultime", "audit complet", "audite ce projet" → 12 étapes
- **Compact** : "audit rapide", "audit léger", "audit compact", "quick audit" → cf. `invocation-guide.md` section "Mode compact"

Si browser dispo → demander viewport(s) souhaité(s) : desktop / mobile / les deux.

---

## STEP 1 — Cartographie initiale

Lecture sans suppositions, dans cet ordre :

1. `README.md`, `CLAUDE.md`, `AGENTS.md`, docs racine
2. `package.json` (scripts, deps, type), `wrangler.toml`, `vite.config.*`, `next.config.*`, configs framework
3. Listing top-level via Glob `**/*` (exclure `node_modules`, `dist`, `build`, `.git`)
4. Entry points : `index.html`, `src/main.*`, `src/index.*`, `pages/`, `app/`, `functions/`
5. Routes/API : `routes/`, `api/`, `functions/api/`, `pages/api/`
6. Persistance : `db/`, `migrations/`, `data/`, `*.sqlite`, JSON stores
7. Auth : recherche `auth`, `jwt`, `session`, `login`, `requireAuth`, `passport`
8. Build / static : `public/`, `dist/`, `static/`

Identifier :
- **Entry points réels** (ce qui charge réellement dans le navigateur)
- **Code probablement mort** (fichiers non importés, `*.bak.*`, modules orphelins)
- **Divergence doc/réalité** (CLAUDE.md dit X, le code fait Y)

**Sortie :** une carte projet de 5 lignes. Pas d'essai littéraire.

---

## STEP 2 — Profil projet

Remplir exactement ce bloc :

```
TYPE:        [static / SPA / SSR / fullstack / API only / hybrid]
STACK:       [HTML+JS / React / Vue / Node+Express / Cloudflare Pages+Workers / etc.]
LAUNCH:      [commande exacte, ex. `npm run dev` → port 8788]
DATA SOURCE: [JSON / D1 / SQLite / API / localStorage / mixed]
CRITICAL:    [3-5 zones : auth, persistence, scanner, dashboard, etc.]
SURFACE:     [nb routes / vues / forms / tables visibles utilisateur]
FRAGILE:     [zones risquées : closure-scoped APIs, monkey-patches, scripts injectés, multi-source-of-truth]
```

Si `LAUNCH` indéterminable depuis `package.json`, demander à l'utilisateur **une seule fois**.

---

## STEP 3 — Lancement local + détection URL

Si projet local + commande lancement :
1. Vérifier serveur déjà actif sur le port (`netstat -an | findstr :PORT`).
2. Sinon : lancer en background via `Bash(run_in_background: true)`.
3. Attendre réponse (`curl -s -o /dev/null -w "%{http_code}" http://localhost:PORT` retourne 200, 401 ou 302).
4. Capturer `BASE_URL = http://localhost:PORT`.

Site statique sans serveur → `python -m http.server 8000` depuis racine static.
URL déployée → utiliser directement.
Aucune URL accessible → continuer en code-only, flagger dans auto-doubt.

---

## STEP 4 — Audit fonctionnel par exécution navigateur

**Cœur de la skill. Exécution réelle. Utilisateur observe.**

Avant chaque test, annoncer : `Test N/M — <nom du scénario>`.

Recettes priorisées dans `playwright-live-testing.md`. Battery par défaut :

1. Cold load (recipe 1) — toujours
2. Auth si présente (recipe 2)
3. Navigation principale (recipe 3)
4. **Persistance d'état actif** (recipe 4) — capture toute la famille de bugs "tab perdu au refresh"
5. CRUD round-trip (recipe 5)
6. Filtres / recherche (recipe 6) — détecte les filtres décoratifs
7. Settings persistence (recipe 7) — détecte saved-mais-jamais-relu
8. Forms (recipe 8)
9. Mobile viewport (recipe 9)
10. Empty state (recipe 10)
11. Network audit (recipe 11)
12. Différentiel (recipe 12)

Après chaque scénario :
- ✅ Pass → note brève
- ❌ Fail → screenshot + console + network logs + hypothèse → marquer comme **suspicion** jusqu'à confirmation par lecture code (step 5)
- ⚠️ Anomalie → comportement partiel → investigation code prévue

**Artefacts** sous `audit-artifacts/<timestamp>/` :
- `screenshot-N-<scenario>.png`
- `console-N.log`
- `network-N.har` (si Playwright)
- `report.md` — log incrémental

---

## STEP 5 — Audit code (Hypothesis-Driven Reading)

Pour chaque anomalie observée navigateur → plonger dans le code pour confirmer la cause racine. Pas de raccourci : un bug UI sans explication code = **suspicion**, pas **confirmé**.

Pour chaque zone critique non couverte par le browser (auth middleware, persistance, store) → lecture ciblée avec patterns connus. Voir `invariants-and-root-cause.md` section "Catalogue des défauts" et le fichier complémentaire `references/detection-patterns.md` (25 signatures de bugs avec recettes Grep).

Patterns plus rentables :
- Closure-scoped APIs jamais exportées
- Monkey-patches sur cibles inexistantes
- Filtres sans logique
- Persistance écrite mais jamais lue
- Defaults qui écrasent les sauvegardes
- Listeners dupliqués
- Vues cachées toujours actives
- Dual source of truth
- Init order races
- Fail-open auth
- API stubs

**Pour chaque pattern confirmé** : citer file:line. Pas de "quelque part dans le code".

---

## STEP 6 — Audit performance (mesures, pas suppositions)

Si browser dispo, mesurer :
- DOM size (`document.querySelectorAll('*').length`) au cold load + après navigation vers chaque vue principale
- Listeners liés (best-effort via DevTools `getEventListeners` si exposé)
- Network waterfall : nb requêtes cold load, total bytes
- Time to first meaningful render (1er élément non-loader visible)

Ne flagger une perf qu'avec mesures, pas avec ressentis. *"DOM 12 450 nœuds au cold load avec 7 onglets tous rendus"* = finding. *"Ça paraît lent"* = pas un finding.

---

## STEP 7 — Audit accessibilité (contrôles concrets)

Sur la page live, pas depuis l'imagination :
- Tabulation clavier sur la nav principale (`keyboard.press('Tab')`). Noter éléments interactifs inatteignables.
- Visibilité du focus (screenshot à chaque position).
- `document.querySelectorAll('button:not([aria-label]):not([title])')` → flagger boutons icône-only sans label.
- Heading order : `Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6')).map(h => h.tagName)`.
- `<img>` sans `alt`.
- Contraste sur le bouton primaire (best-effort, flag si visiblement faible).

Pas d'essai WCAG générique. Reporter uniquement ce qui échoue.

---

## STEP 8 — Audit sécurité (ciblé, pas spéculatif)

Vérifier ces patterns par défaut. Élargir seulement si contexte le justifie :
- Bypass routes — recherche `bypass`, `dev-login`, `__test`, `?admin=`, `localhost-only`
- Secrets dans le repo — `.env` commit, clés API hardcodées, `password = "..."` (Grep ciblé)
- Fail-open auth — `try { authenticate() } catch { /* swallow */ next() }`
- Routes debug oubliées — `/api/debug`, `/api/dump`, `/api/all`, `/_admin`
- localStorage avec données sensibles — JWTs, mots de passe, objets user complets

Ne pas fabriquer de risques. Si rien : *"Sécurité : aucun pattern à risque détecté lors du scan ciblé."*

---

## STEP 9 — Audit données / persistance (invariants métier)

Identifier les invariants depuis le profil (step 2). Voir `invariants-and-root-cause.md` pour la méthode. Pour chaque invariant :
- 1 scénario navigateur
- l'exécuter
- confirmer ou violer
- violation = bug confirmé

Puis lecture code pour confirmer cause racine. Patterns à chercher :
- Patch spreads dans le mauvais ordre (`{...defaults, ...patch}` vs `{...patch, ...defaults}`)
- Read avec default hardcodé qui masque la valeur sauvegardée
- Write avec payload tronqué (sauve seulement certains champs)
- Save fire mais read avant resolve (race)

---

## STEP 10 — Tentative de réparation (auto-fix sous contrôle)

Voir `repair-rules.md` pour le protocole complet. Résumé :

1. Isoler à une cause racine
2. Définir l'invariant restauré (1 phrase)
3. Classifier : SAFE / SAFE+REGRESSION / MEDIUM / HIGH-RISK
4. Modifier le minimum (pas de cleanup opportuniste)
5. Commit avec message qui nomme l'invariant (`fix(<area>): <invariant>`)
6. Re-run le scénario qui exposait le bug
7. Re-run 1-2 scénarios adjacents

**Stop à 3 fixes par audit** sauf demande explicite — préserve le budget de revue cognitive de l'utilisateur.

---

## STEP 11 — Revalidation (obligatoire après tout fix)

Pour chaque fix appliqué :
- Re-run scénario d'origine (doit passer)
- Run 1-2 scénarios adjacents (doivent toujours passer)
- Pour fixes UI/persistance : reload + re-test (l'état doit persister)
- Capturer screenshots avant/après si visible

Régression détectée → **revert immédiat** + déclasser le fix en "Risky, postpone".

---

## STEP 12 — Rapport final priorisé

Voir `report-template.md` pour le squelette exact. Sections obligatoires :
- Résumé exécutif
- Profil projet
- Méthode utilisée
- Cartographie réelle
- Ce qui fonctionne réellement
- Bugs confirmés (template par finding : `finding-schema.md`)
- Suspicions à vérifier
- Illusions de fonctionnalité
- Performance mesurée
- Accessibilité (échecs only)
- Sécurité / robustesse
- Dette technique
- Correctifs appliqués
- Régressions détectées ou non
- Score global
- Plan d'attaque (Top 3 P0, quick wins, dette, surveillance)
- Niveau de confiance global
- Risques résiduels avant merge
- Auto-doubt (`anti-false-positives.md`)

---

## Allocation d'énergie d'audit

Ordre de priorité par étape s'il faut couper court :

1. Flux critiques métier
2. Persistance / cohérence des données
3. Navigation et état actif
4. Paramètres / configuration
5. Performance réelle
6. Accessibilité critique
7. Dette structurelle

Ne pas perdre du temps sur des détails cosmétiques tant que des flux critiques sont possiblement faux.
