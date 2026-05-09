# Report Template — Audit Ultime

The exact skeleton to fill at step 12. Do not improvise structure. Empty sections are acceptable — write "Aucun élément détecté" or "N/A". Never delete sections.

---

```markdown
# Audit Ultime — <project name> — <YYYY-MM-DD>

## Résumé exécutif

- **Bugs confirmés:** N (dont X P0, Y P1)
- **Suspicions à vérifier:** N
- **Illusions de fonctionnalité:** N
- **Quick wins:** N
- **Correctifs appliqués:** N (X safe, Y avec validation)
- **Régressions détectées:** N
- **Niveau de confiance global:** [Haut / Moyen / Bas] — <one-sentence justification>

---

## Profil du projet

```
TYPE:        ...
STACK:       ...
LAUNCH:      ...
DATA SOURCE: ...
CRITICAL:    ...
SURFACE:     ...
FRAGILE:     ...
```

---

## Méthode d'audit utilisée

**Outils navigateur:** [Playwright MCP / Browser MCP / local Playwright / aucun]
**Viewports testés:** [desktop 1280×800 / mobile 375×812 / autre]
**Scénarios exécutés:** N (sur M planifiés — voir liste détaillée plus bas)
**Durée audit:** ~X min
**Artefacts produits:** `audit-artifacts/<timestamp>/` — N screenshots, N traces, N logs

**Scénarios exécutés (statut):**

| # | Scénario | Statut | Preuve |
|---|----------|--------|--------|
| 1 | Cold load + console capture | ✅ / ❌ / ⚠️ | screenshot-01.png |
| 2 | Auth flow | ✅ | — |
| 3 | Main navigation | ❌ (3/7 onglets cassés) | screenshots/nav-*.png |
| ... | ... | ... | ... |

---

## Cartographie réelle

5–10 lines describing the actual project structure, entry points, data flow, and notable divergences from documentation.

**Divergences doc/réalité:**
- <list, or "Aucune">

**Code probablement mort:**
- `path/to/file.bak.js` — <reason>
- ...

---

## Ce qui fonctionne réellement

| Fonctionnalité | Statut | Preuve |
|----------------|--------|--------|
| Login | ✅ Réellement fonctionnelle | Scénario 2 + persistance session confirmée |
| Navigation principale | ⚠️ Fragile | 4/7 onglets OK, 3 ne se chargent pas |
| CRUD cartouches | ✅ | Scénario 5 — create+reload+edit+delete OK |
| ... | ... | ... |

---

## Bugs confirmés

### B1 — <Titre court et précis>

| Field | Value |
|-------|-------|
| Catégorie | <fonctionnel/UI/perf/a11y/sécurité/données/archi> |
| Gravité | <Critique/Haute/Moyenne/Faible> |
| Confiance | Confirmé |
| Reproductibilité | <Toujours/Intermittent> |
| Portée | <Localisée/Multi-vues/Systémique> |
| Impact utilisateur | <Bloquant/Gênant/Mineur/Invisible> |
| Coût correction | <Faible/Moyen/Élevé> |
| Priorité | <P0/P1/P2/P3> |
| Statut | <confirmé non corrigé / corrigé / partiel> |

**Où observé:** <view name + URL or file:line>

**Reproduction:**
1. <step>
2. <step>
3. <step>

**Résultat observé:** ...
**Résultat attendu:** ...

**Cause racine:** <one sentence + file:line>

**Trust boundary:** <front↔back / UI↔state / config↔runtime / etc.>

**Preuve:**
- Screenshot: `audit-artifacts/.../bug-B1.png`
- Console:
  ```
  <log excerpt>
  ```
- Code (file:line):
  ```js
  <excerpt showing the defect>
  ```

**Proposition de correction:** <patch in 3-5 lines>

**Risque correction:** <Safe / Safe with validation / Risky / Architecture change>

---

### B2 — ...

(repeat per confirmed bug)

---

## Suspicions à vérifier

For each suspicion: same template as confirmed bugs but **Confiance: Très probable / Possible** and **Statut: à investiguer**. Add a "Comment confirmer" section.

### S1 — <Titre>

...

**Comment confirmer:**
- <action user can take, or test that needs running with credentials we don't have>

---

## Illusions de fonctionnalité

UI promises a feature; reality says no. Listed prominently because trust impact is high.

### I1 — <Feature name>

**Promesse UI:** <what the user sees and expects>
**Réalité:** <what actually happens / doesn't happen>
**Preuve:** <code excerpt + browser observation>
**Recommandation:** Soit (a) implémenter réellement, soit (b) retirer l'élément UI trompeur.

---

## Performance (mesures réelles)

| Mesure | Valeur observée | Seuil acceptable | Verdict |
|--------|-----------------|------------------|---------|
| DOM nodes au cold load | 12,450 | <2,000 | ❌ |
| Requêtes au cold load | 47 | <30 | ⚠️ |
| Time to first meaningful render | 1.8s | <1.5s | ⚠️ |
| Bytes transférés (cold) | 850 KB | <500 KB | ⚠️ |
| ... | ... | ... | ... |

**Causes mesurées (non spéculées):**
- ...

---

## Accessibilité (échecs uniquement)

Liste seulement ce qui a échoué aux contrôles ciblés (step 7). Ne pas paraphraser WCAG.

- **Tab navigation:** 3 boutons inatteignables au clavier (`button[data-fab]`, ...) — file:line
- **Boutons icône-only sans label:** 5 trouvés — `#notif-btn`, `#logout-btn`, ... — sans `aria-label` ni `title`
- **Contraste insuffisant:** bouton primaire `.btn-cancel` — couleur sur fond — ratio ~2.8 (min 4.5)

Si rien à signaler: "Contrôles a11y ciblés passés."

---

## Sécurité / robustesse

Findings ciblés (step 8). Pas de FUD.

- ...

Si rien: "Aucun pattern à risque détecté lors du scan ciblé. Audit complet hors scope."

---

## Dette technique

Pas un bug, pas urgent. Rendra les bugs futurs plus probables.

- `public/index.html` (~7400 lignes) — fichier monolithique. Recommandation: split par responsabilité dans une phase planifiée.
- ...

---

## Correctifs appliqués

### F1 — <fix title>

**Bug corrigé:** B<id>
**Classification:** <Safe / Safe with validation>
**Modification:** <one-line summary + file:line>
**Diff:**
```diff
- ancien code
+ nouveau code
```
**Re-test:** Scénario <N> rejoué — ✅ pass
**Adjacents vérifiés:** Scénarios <N>, <M> — ✅ pas de régression
**Avant/après:**
- avant: <screenshot or behavior>
- après: <screenshot or behavior>

---

### F2 — ...

---

## Régressions détectées ou non

| Fix | Scénarios re-testés | Régression ? | Détails |
|-----|---------------------|--------------|---------|
| F1 | 3, 5, 12 | Non | — |
| F2 | 4, 7 | Oui — sur 7 | Détail: ... → fix retiré, marqué "Risky postpone" |

---

## Score global du projet

| Axe | Score /5 | Justification |
|-----|----------|---------------|
| Fiabilité fonctionnelle | 3 | Flux principaux OK, tab persistence cassée jusqu'à F1 |
| Cohérence métier | 4 | Invariants stock respectés sauf un cas multi-entrepôt |
| Robustesse UI | 3 | Plusieurs illusions de fonctionnalité (filtres) |
| Performance perçue | 2 | DOM trop gros, tous les onglets rendus au load |
| Accessibilité | 2 | 5 boutons icône sans label, focus invisible |
| Maintenabilité | 2 | index.html ~7400 lignes, dette structurelle |
| Confiance audit | 4 | 11/12 scénarios browser exécutés, 25 patterns vérifiés |

---

## Plan d'attaque

### Top 3 P0 — corriger immédiatement

1. **<Bug ID + title>** — <1-line why critical> — coût: <faible/moyen/élevé>
2. ...
3. ...

### Top 3 quick wins

1. ...
2. ...
3. ...

### Top 3 dette à planifier

1. ...
2. ...
3. ...

### Top 3 zones à surveiller

1. ...
2. ...
3. ...

### Tests de non-régression à ajouter

- Scénario `tab-persistence-after-reload` (recipe 4) — à intégrer en CI
- ...

### Risques à vérifier avant merge / déploiement

- ...

---

## Auto-doubt

**Réellement vérifié:**
- <list scenarios run live with passing/failing result>
- <list invariants probed>

**Vérifié uniquement par lecture de code:**
- <list zones where browser test was not possible>

**Non vérifiable dans ce contexte:**
- <list zones requiring credentials, prod data, etc.>

**Hypothèses non invalidées:**
- <list suspicions that survived but lack proof>

**Risques de faux négatifs:**
- <list known blind spots: viewports not tested, flows not run, edge cases skipped>

---

## Fichiers modifiés (si correctifs appliqués)

```
public/index.html       (4 lignes modifiées)
public/js/foo.js        (1 fonction ajoutée)
```

## Commit conseillé

```
fix(<area>): <invariant restored>

- B1: <bug name> → F1 <fix description>
- B3: <bug name> → F2 <fix description>

Tested: scenarios 3, 5, 12 re-run. No regression on adjacent flows 4, 7.
```

## PR conseillée

**Titre:** `fix: audit ultime — N P0 bugs corrigés`

**Body:**
```
## Summary
Issues fixed during audit-ultime session on <date>:
- B1: <one-line>
- B3: <one-line>

## Test Plan
- [ ] Reproduce B1 steps — confirm fixed
- [ ] Reproduce B3 steps — confirm fixed
- [ ] Run scenarios 4, 7 (adjacent flows) — no regression
- [ ] Mobile viewport check on affected views

## Out of scope (audit findings, not fixed here)
- B2 (Risky, postpone) — needs design discussion
- Performance debt (DOM size) — planned separately
- ...
```

---

End of report.
```
