# Report Template — Audit Ultime

Squelette exact à remplir au step 12. Ne pas improviser la structure. Sections vides acceptées avec "Aucun élément détecté" ou "N/A". Ne jamais supprimer de section.

---

```markdown
# Audit Ultime — <project name> — <YYYY-MM-DD>

## Résumé exécutif

- **Bugs confirmés:** N (dont X P0, Y P1)
- **Suspicions à vérifier:** N
- **Illusions de fonctionnalité:** N
- **Quick wins:** N
- **Correctifs appliqués:** N (X safe, Y avec validation, Z partiel)
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

5–10 lignes décrivant la structure projet réelle, entry points, data flow, divergences notables vs documentation.

**Divergences doc/réalité:**
- <list, ou "Aucune">

**Code probablement mort:**
- `path/to/file.bak.js` — <raison>
- ...

---

## Ce qui fonctionne réellement

| Fonctionnalité | Statut | Preuve |
|----------------|--------|--------|
| Login | ✅ Réellement fonctionnelle | Scénario 2 + persistance session confirmée |
| Navigation principale | ⚠️ Fragile | 4/7 onglets OK, 3 ne se chargent pas |
| CRUD cartouches | ✅ | Scénario 5 — create+reload+edit+delete OK |
| ... | ... | ... |

(Symboles autorisés : ✅ ⚠️ 🔶 ❌ ❓ 💥 🛠 👀 — cf. matrice de vérité dans `invariants-and-root-cause.md`.)

---

## Bugs confirmés

(Format par finding : voir `finding-schema.md`. Reproduit ici pour référence rapide.)

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

**Où observé:** <vue + URL ou file:line>

**Reproduction:**
1. <step>
2. <step>

**Résultat observé:** ...
**Résultat attendu:** ...

**Cause racine:** <une phrase + file:line>

**Trust boundary:** <front↔back / UI↔state / config↔runtime / etc.>

**Preuve:**
- Screenshot: `audit-artifacts/.../bug-B1.png`
- Console: ```<log>```
- Code (file:line): ```<excerpt>```

**Proposition de correction:** <patch en 3-5 lignes>

**Risque correction:** <SAFE / SAFE+REGRESSION / MEDIUM / HIGH-RISK>

---

### B2 — ...

(répéter par bug confirmé)

---

## Suspicions à vérifier

Même template que les bugs confirmés mais **Confiance: Très probable / Possible** et **Statut: à investiguer**. Ajouter une section "Comment confirmer".

### S1 — <Titre>

...

**Comment confirmer:**
- <action user / test à lancer avec creds qu'on n'a pas>

---

## Illusions de fonctionnalité

UI promet, réalité non. Listées séparément car l'impact sur la confiance utilisateur est élevé.

### I1 — <Feature name>

**Promesse UI:** <ce que l'utilisateur voit et attend>
**Réalité:** <ce qui se passe / ne se passe pas>
**Preuve:** <code excerpt + observation browser>
**Recommandation:** Soit (a) implémenter réellement, soit (b) retirer l'élément UI trompeur.

---

## Performance (mesures réelles)

| Mesure | Valeur observée | Seuil acceptable | Verdict |
|--------|-----------------|------------------|---------|
| DOM nodes au cold load | 12 450 | < 2 000 | ❌ |
| Requêtes au cold load | 47 | < 30 | ⚠️ |
| Time to first meaningful render | 1.8 s | < 1.5 s | ⚠️ |
| Bytes transférés (cold) | 850 KB | < 500 KB | ⚠️ |
| ... | ... | ... | ... |

**Causes mesurées (pas spéculées):**
- ...

---

## Accessibilité (échecs uniquement)

Lister seulement ce qui a échoué aux contrôles ciblés (step 7). Ne pas paraphraser WCAG.

- **Tab navigation:** 3 boutons inatteignables au clavier (`button[data-fab]`, ...) — file:line
- **Boutons icône-only sans label:** 5 trouvés — `#notif-btn`, `#logout-btn`, ... — sans `aria-label` ni `title`
- **Contraste insuffisant:** bouton `.btn-cancel` — ratio ~2.8 (min 4.5)

Si rien : "Contrôles a11y ciblés passés."

---

## Sécurité / robustesse

Findings ciblés (step 8). Pas de FUD.

- ...

Si rien : "Aucun pattern à risque détecté lors du scan ciblé. Audit complet hors scope."

---

## Dette technique

Pas un bug, pas urgent. Augmente la probabilité des bugs futurs.

- `public/index.html` (~7 400 lignes) — fichier monolithique. Recommandation : split par responsabilité dans une phase planifiée.
- ...

---

## Correctifs appliqués

(Voir `repair-rules.md` pour les classes et le protocole de validation.)

### F1 — <fix title>

**Bug corrigé:** B<id>
**Classification:** <SAFE / SAFE+REGRESSION / MEDIUM>
**Modification:** <one-line summary + file:line>

**Diff:**
```diff
- ancien code
+ nouveau code
```

**Avant:** <symptôme observé>
**Correction:** <ce qui a été modifié>
**Après:** <comportement après revalidation>
**Vérifications associées:** Scénario <N> rejoué — ✅ ; scénarios voisins <X>, <Y> — ✅
**Confiance:** élevée / moyenne / partielle

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
| Maintenabilité | 2 | index.html ~7 400 lignes, dette structurelle |
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

---

## Niveau de confiance global

[Haut / Moyen / Bas] — <justification en une phrase, citant le % de scénarios live exécutés et les zones non couvertes>

---

## Correctifs sûrs appliqués

Liste des fixes classés SAFE ou SAFE+REGRESSION, avec confiance **élevée** dans le bloc avant/après.

- F1 — ...
- F3 — ...

---

## Correctifs appliqués mais à revalider

Fixes avec confiance **moyenne** ou **partielle**. Préciser ce qui n'a pas été couvert.

- F2 — appliqué, mais validation mobile non possible (browser tool absent en mobile viewport)
- ...

---

## Correctifs non appliqués car trop risqués

Bugs confirmés dont le fix tombe en classe **HIGH-RISK** (ou MEDIUM sans accord utilisateur). Donner :
- Le bug ID
- Le patch proposé
- La raison du report (architecture, contrat, dépendance, ampleur)

- B5 — patch proposé : <…> — non appliqué car nécessite migration de schéma localStorage
- ...

---

## Régressions détectées après fix

(Combiné avec le tableau plus haut, ou détaillé ici si plusieurs régressions à expliquer.)

- Aucune détectée dans le périmètre testé.
- OU : F2 a cassé le scénario 7 (mobile nav) → fix retiré.

---

## Régressions non détectées sur le périmètre testé

Lister les scénarios re-joués post-fix qui n'ont pas révélé de régression. Permet à l'utilisateur de juger ce qui a effectivement été couvert.

- Scénarios 3, 4, 5, 12 sur desktop : rejoués post-F1, F2, F3 — pas de régression
- Scénario 9 (mobile) : non rejoué post-F2 → faux négatif possible

---

## Zones non couvertes

Listing explicite des zones non testées dans cette session :
- <feature / vue / flux> — raison
- ...

---

## Risques résiduels avant merge / déploiement

Liste ordonnée par criticité. Section obligatoire **même si vide**.

- P0 : <risque>
- P1 : <risque>
- ...

OU : "Aucun risque résiduel identifié dans le périmètre testé."

---

## Auto-doubt

(Bloc obligatoire — cf. `anti-false-positives.md`.)

**Réellement vérifié:**
- <scénarios qui ont tourné en navigateur avec résultat pass/fail>
- <invariants probés>

**Vérifié uniquement par lecture de code:**
- <zones où le browser test n'a pas été possible>

**Non vérifiable dans ce contexte:**
- <zones nécessitant creds, prod data, accès externe>

**Hypothèses non invalidées:**
- <suspicions qui ont survécu mais manquent de preuve>

**Risques de faux négatifs:**
- <blind spots : viewports non testés, flux non joués, edge cases skippés>

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
- B2 (HIGH-RISK postpone) — needs design discussion
- Performance debt (DOM size) — planned separately
- ...
```

---

End of report.
```
