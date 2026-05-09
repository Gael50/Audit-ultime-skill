# Triage et sévérité — Audit Ultime

Comment trier, scorer et prioriser les findings. Les règles ici déterminent ce qui apparaît dans le Top 3 P0 du rapport final.

---

## Matrice de gravité

| Gravité | Définition | Exemples |
|---------|------------|----------|
| **Critique** | Bloque un flux critique, perte/corruption de données, faille de sécurité, illusion sur une fonctionnalité métier centrale | Auth fail-open, save qui n'enregistre pas, suppression qui réapparaît, filtre métier décoratif sur un flux à enjeu |
| **Haute** | Gêne forte récurrente, persistance fragile, dette de confiance, perf visiblement dégradée | État onglet perdu au refresh, listener dupliqué qui crée des doublons, settings sauvegardés mais non relus |
| **Moyenne** | Gêne ponctuelle, écart visuel notable, accessibilité partielle, code obsolète actif | Bouton icône sans label, vue mobile cassée sur un cas, perf passable sur grosse liste |
| **Faible** | Cosmétique, dette mineure, edge case improbable | Console.log oublié, classe CSS inutilisée, un alt texte manquant sur image décorative |

Gravité = **impact intrinsèque** du bug. Pas la priorité.

---

## Niveaux de confiance

| Confiance | Conditions |
|-----------|-----------|
| **Confirmé** | Reproduction stable navigateur OU runtime ET cause racine identifiée à file:line |
| **Très probable** | Un des deux fortement présent : reproduction sans cause racine claire, ou cause racine évidente sans repro live |
| **Possible** | Indices code/UI ou hypothèse cohérente, pas encore invalidée |

Règle :
- **Confirmé** → va dans "Bugs confirmés"
- **Très probable** + **Possible** → vont dans "Suspicions à vérifier"

Pas d'inflation : une suspicion forte n'est pas un confirmé.

---

## Reproductibilité

| Niveau | Signification | Action |
|--------|---------------|--------|
| Toujours | 100 % avec mêmes étapes | Bug solide, prioritaire |
| Intermittent | Fraction des essais (noter le taux : "3/5") | Investigation race / état initial |
| Non confirmé | Observé une fois, non reproduit | Garde en surveillance, pas en P0 |

---

## Calcul de priorité (P0–P3)

La priorité n'est pas la gravité. Elle combine 7 facteurs :

```
priorité = f(impact_user, fréquence, risque_métier, risque_régression, coût_correction, confiance_diagnostic, visibilité)
```

### Décision rapide

| Si | Alors |
|----|-------|
| Critique + Bloquant + Confirmé + Coût ≤ Moyen | **P0** |
| Critique + Confirmé + Coût Élevé | **P1** (sauf si data loss → P0) |
| Haute + Gênant + Confirmé + Coût Faible | **P0** ou **P1** selon urgence |
| Haute + Confirmé + Coût Élevé | **P1** |
| Moyenne + Confirmé + Coût Faible | **P2** |
| Moyenne + Possible + Coût Moyen | **P3** |
| Faible + tout | **P3** |
| Très probable + bloquant | **P1** (jamais P0 sans preuve) |
| Possible | **P3** maximum |

### Règle de combinaison

Toute matrice est une heuristique. Cas limites :
- **Data loss** → toujours P0, quelle que soit la confiance
- **Sécurité auth** → P0 si confiance ≥ Très probable
- **Perte de fonctionnalité documentée comme stable** → P0 si confirmée

---

## Catégories de triage opérationnel

Au-delà des P0/P1/P2/P3, organiser les findings par action :

### A. Corriger immédiatement
- Bug bloquant
- Perte / corruption de données
- Faille sécurité confirmée
- Incohérence métier grave
- Illusion de fonctionnalité critique

### B. Corriger ce lot si fix faible risque
- Bug fréquent gênant
- Persistance partielle
- Perf évidente avec correctif local
- Accessibilité cassée sur composant central

### C. Planifier
- Dette structurelle
- Refactor utile non urgent
- Simplification d'architecture
- Optimisation profonde

### D. Surveiller
- Odeurs faibles
- Suspicions non confirmées
- Cas rares
- Bugs intermittents non reproduits

Cette taxonomie remplit les sections "Plan d'attaque" du rapport.

---

## Règles spécifiques aux illusions de fonctionnalité

Une **illusion** = l'UI promet une fonctionnalité, la réalité dit non.

Toute illusion confirmée monte d'un cran de gravité par rapport à un bug équivalent qui ne ment pas à l'utilisateur, parce que :
- Elle érode la confiance utilisateur
- Elle masque le défaut aux yeux du dev
- Elle complique la priorisation interne (le bug n'est pas signalé puisque "ça a l'air de marcher")

Donc :
- Filtre décoratif sur flux critique → **Critique**, jamais Moyenne
- Bouton qui ne fait rien sur action métier → **Critique**
- Onglet qui charge mais ne montre pas les bonnes données → **Haute** minimum

Le rapport doit avoir une section dédiée "Illusions de fonctionnalité" même si c'est juste pour dire "Aucune détectée".

---

## Score global du projet

À calculer en fin d'audit. Notation /5 par axe, avec justification courte.

| Axe | Que mesurer |
|-----|-------------|
| Fiabilité fonctionnelle | Flux critiques qui passent vs cassent |
| Cohérence métier | Invariants respectés / violés |
| Robustesse UI | Présence d'illusions, fragilité au refresh, mobile |
| Performance perçue | DOM, requêtes, render, perçu utilisateur |
| Accessibilité | Échecs aux contrôles ciblés |
| Maintenabilité | Taille fichiers, duplication, dette |
| Confiance audit | % de scénarios live vs code-only, % patterns vérifiés |

Le score n'est pas un classement absolu. C'est une boussole pour piloter les priorités.

---

## Garde-fou : l'inflation

Les rapports d'audit dérivent vers l'inflation : trop de findings, trop de "critique", peu d'action. Pour rester utile :

- Si plus de **5 P0** → recombiner les findings qui partagent une cause racine
- Si plus de **15 findings totaux** → vérifier que les "Faibles" méritent vraiment leur place
- Une "dette technique" listée n'est pas un finding ; elle va dans la section Dette
- Une "amélioration possible" n'est pas un bug ; elle ne va dans le rapport que si l'utilisateur l'a demandée

Le but est l'action, pas la couverture exhaustive.

---

## Garde-fou : le sous-triage

Inverse : ne pas minimiser pour faire joli.

- Si l'auth est fail-open et que c'est confirmé → **P0**, ne pas adoucir
- Si une feature critique est une illusion → ne pas la déclasser en "à améliorer"
- Si un fix risquerait une régression → le déclasser à **HIGH-RISK** dans la classification fix, mais garder le bug à sa vraie gravité

Le triage des bugs et la classification des fixes sont indépendants. Un bug critique peut nécessiter un fix HIGH-RISK qu'on n'auto-applique pas — le rapport doit dire les deux.
