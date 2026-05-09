# Finding Schema — Audit Ultime

Format obligatoire pour chaque bug confirmé ou suspicion significative. Utilisé dans la section "Bugs confirmés" et "Suspicions" du rapport final.

Ne pas improviser de format. Un finding sans ce squelette ne compte pas.

---

## Template par finding

```markdown
### [TITRE COURT ET PRÉCIS]

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

**Où observé:** <vue + URL ou file:line>

**Reproduction:**
1. <étape précise>
2. ...
3. ...

**Résultat observé:** <ce qui se passe>
**Résultat attendu:** <ce qui devrait se passer>

**Cause racine:** <une phrase, file:line si connue>

**Trust boundary:** <front↔back / UI↔state / config↔runtime / etc.>

**Preuve:**
- Screenshot: `audit-artifacts/.../bug-XX.png`
- Console:
  ```
  <log excerpt>
  ```
- Code (file:line):
  ```js
  <excerpt montrant le défaut>
  ```

**Proposition de correction:** <patch description ≤5 lignes>

**Risque correction:** <Safe / Safe with validation / Medium / High-Risk> (cf. `repair-rules.md`)
```

---

## Champs obligatoires

Aucun finding ne doit être livré sans ces 13 champs renseignés. Si un champ est inconnu :
- "Non confirmé" pour Reproductibilité
- "—" pour Cause racine si impossible à isoler (le finding redescend alors automatiquement en **Confiance: Possible**)
- "À déterminer après diagnostic" pour Coût et Risque correction

**Jamais de champ vide ni de placeholder TODO.**

---

## Règles de calcul

### Gravité
- **Critique** — bloque un flux critique, perte de données, faille de sécurité, illusion sur fonctionnalité métier centrale
- **Haute** — gêne forte, dette de confiance, persistance fragile, perf visiblement dégradée
- **Moyenne** — gêne ponctuelle, écart visuel, accessibilité partielle
- **Faible** — détail cosmétique, dette mineure

### Confiance
- **Confirmé** — reproduction stable + cause racine identifiée à file:line
- **Très probable** — un des deux manque mais l'autre est très solide (reproduction sans cause racine identifiée, ou cause racine évidente sans reproduction réelle)
- **Possible** — indices, hypothèse non invalidée, à investiguer

Voir `anti-false-positives.md` pour la règle d'or "pas de Confirmé sans preuve duale".

### Priorité (P0/P1/P2/P3)

La priorité **n'est pas** la gravité. Elle se calcule à partir de :

```
priorité = f(impact × fréquence × risque_métier × risque_régression × coût_correction × confiance_diagnostic)
```

Tableau de décision rapide :

| Gravité | Impact | Confiance | Coût fix | Priorité |
|---------|--------|-----------|----------|----------|
| Critique | Bloquant | Confirmé | Faible | **P0** |
| Critique | Bloquant | Très probable | Faible/Moyen | **P0** |
| Haute | Gênant | Confirmé | Faible | **P0** ou **P1** |
| Haute | Gênant | Confirmé | Élevé | **P1** |
| Moyenne | Mineur | Confirmé | Faible | **P2** |
| Moyenne | Mineur | Possible | Moyen | **P3** |
| Faible | Invisible | * | * | **P3** |

Voir `triage-and-severity.md` pour la matrice complète + cas limites.

### Reproductibilité
- **Toujours** — reproductible à 100 % avec les mêmes étapes
- **Intermittent** — se produit dans une fraction des essais → noter le taux observé (ex. "3 fois sur 5")
- **Non confirmé** — observé une fois, pas reproduit ensuite

### Portée
- **Localisée** — un fichier, une fonction, une vue
- **Multi-vues** — touche plusieurs UI views
- **Systémique** — touche un invariant transverse, plusieurs modules

---

## Statut final

| Statut | Signification |
|--------|---------------|
| confirmé non corrigé | bug prouvé, fix non appliqué (trop risqué, hors scope, ou postponed) |
| corrigé | fix appliqué + revalidation complète (cf. `repair-rules.md`) |
| corrigé à confirmer | fix appliqué mais protocole de revalidation incomplet |
| correctif appliqué, validation incomplète | équivalent ; choisir une formulation par audit, pas mélanger |
| partiel | fix mitige le symptôme mais ne traite pas la cause racine |
| à investiguer | suspicion non confirmée |
| non reproductible | observé une fois, ne se reproduit plus → garder en surveillance |

---

## Règle d'or

Un finding sans **preuve** (screenshot, log, ou code excerpt avec file:line) **n'est pas** un finding. Il est une hypothèse.

Une hypothèse va dans la section "Suspicions à vérifier", pas dans "Bugs confirmés".
