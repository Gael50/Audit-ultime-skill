# Anti-Faux Positifs — Audit Ultime

Comment réduire agressivement les faux positifs et reconnaître honnêtement les faux négatifs. Un rapport d'audit qui crie "danger" sur 50 zones sans preuve est pire qu'un rapport qui ne dit rien.

Le rapport doit distinguer clairement : **confirmé / probable / incertain**.

---

## Règle d'or : pas de "Confirmé" sans preuve duale

Un finding ne porte le statut **Confiance: Confirmé** que si **au moins un** des éléments suivants existe :

1. **Reproduction réelle** dans un scénario navigateur ou runtime
2. **Lecture de code** démontrant clairement le défaut, avec file:line
3. **Erreur observable** dans la console, le réseau, ou le runtime
4. **Incohérence prouvée** entre UI, état et persistance (mesurée, pas supposée)
5. **Démonstration** que la feature affichée est en réalité non branchée

Sans aucun de ces éléments, le finding redescend automatiquement vers une catégorie inférieure :
- Suspicion forte (Très probable)
- Suspicion faible (Possible)
- Dette technique (pas un bug)
- Piste à vérifier (pas de claim)

---

## Anti-patterns à bannir absolument

Ces formulations ne doivent jamais apparaître dans un rapport :

- "Il semble que…" → soit prouve, soit reclasse en suspicion
- "Probablement…" sans contexte d'investigation → précise le niveau de probabilité
- "Cela pourrait causer…" → ou prouve que ça cause, ou retire le claim
- "Il faudrait vérifier…" → c'est ton job, vérifie maintenant ou flagge en suspicion explicite
- "Risque potentiel" sans estimation → ou estime, ou retire
- "Best practice non respectée" sans impact mesuré → ne va pas dans le rapport sauf demande explicite
- Recommandations génériques copiées d'un blog post → ne va pas dans le rapport

---

## Fond commun pour qualifier un finding

Avant de l'inclure, vérifier ce trio :

1. **L'utilisateur en est-il affecté concrètement ?** Si non → ne va pas dans P0/P1
2. **Ai-je une preuve que je peux montrer ?** Si non → suspicion, pas confirmé
3. **Ai-je au moins un fichier:ligne ou un screenshot ?** Si non → ne va pas dans "Bugs confirmés"

Si une seule réponse est non → reclassifier ou retirer.

---

## Différence entre les niveaux

### Confirmé
- Reproduction stable + cause racine à file:line
- Va dans la section "Bugs confirmés" du rapport
- Peut être P0/P1/P2/P3 selon impact

### Très probable
- Reproduction stable mais cause racine pas encore identifiée
- OU cause racine évidente sans repro live (ex. fail-open auth visible dans le code)
- Va dans "Suspicions à vérifier"
- Maximum P1, jamais P0 sauf data loss / sécurité

### Possible
- Indices code/UI cohérents
- Hypothèse non invalidée
- Va dans "Suspicions à vérifier" avec section "Comment confirmer"
- Maximum P2

### Piste / Dette
- Pas un bug
- Va dans "Dette technique" si structurel, ou est retiré sinon

Ne pas confondre ces niveaux. L'inflation tue la crédibilité du rapport.

---

## Réduction des faux positifs

### Croiser les sources

Un finding solide combine au minimum 2 sources sur 3 :
- Comportement observé (browser test)
- Code lu (file:line)
- Persistance vérifiée (localStorage, DB, network response)

Un seul de ces signaux = suspicion. Deux = confirmé en général.

### Tenir compte du contexte projet

Pas tout pattern est un bug :
- Un stub peut être intentionnel (placeholder MVP)
- Un duplicate peut être un fallback délibéré
- Un default qui semble écraser peut être un comportement choisi
- Un console.log peut être un debug aid intentionnel en dev

Avant de flagger : lire CLAUDE.md, README, commit messages récents. Si c'est documenté comme intentionnel → ne pas flagger comme bug.

### Vérifier le comportement réel

Le code peut sembler buggé sans qu'un bug soit réellement déclenché en production. Inversement, du code "propre" peut produire un bug par interaction. Toujours vérifier l'effet observable plutôt que de juger sur le code seul.

---

## Reconnaissance honnête des faux négatifs

Un audit ne peut pas tout couvrir. Le rapport doit lister explicitement ce qui n'a pas été testé :

- Zones non explorées (manque de temps, scope volontairement restreint)
- Chemins non accessibles (auth-gated avec creds manquants, prod data, branches cachées de l'app)
- Dépendances absentes (browser MCP indisponible, base de données pas montée, API externe inaccessible)
- Comportements intermittents (testés mais non reproduits dans cette session)
- Scénarios utilisateur non couverts (edge cases, mobile orientation, slow network)

Sans ce listing → l'audit prétend une couverture qu'il n'a pas → faux négatifs invisibles.

---

## Section auto-doubt obligatoire

À la fin du rapport, **toujours** inclure ce bloc, sans exception :

```markdown
## Auto-doubt

**Réellement vérifié:**
- <scénarios qui ont tourné en navigateur avec résultat pass/fail>
- <invariants probés>

**Vérifié uniquement par lecture de code:**
- <zones où le browser test n'a pas été possible>

**Non vérifiable dans ce contexte:**
- <zones nécessitant des creds, prod data, accès externe>

**Hypothèses non invalidées:**
- <suspicions qui ont survécu mais manquent de preuve>

**Risques de faux négatifs:**
- <blind spots connus : viewports non testés, flux non joués, edge cases skippés>
```

Cette section transforme l'audit d'un "verdict" en un "instantané traçable". Elle protège l'utilisateur contre les conclusions trop fortes.

---

## Auto-vérification interne

Avant de figer le rapport, se poser :

- Qu'ai-je vraiment prouvé ?
- Qu'ai-je seulement inféré ?
- Où suis-je le plus susceptible de me tromper ?
- Quelles hypothèses n'ont pas été invalidées ?
- Quels faux négatifs sont possibles vu ce que j'ai testé ?
- Mon Top 3 P0 est-il défendable au mot près ?

Si la skill ne peut pas répondre à ces 6 questions → le rapport n'est pas prêt.

---

## Honnêteté sur les correctifs

Pour les fixes appliqués, le rapport distingue :

| Statut | Conditions |
|--------|------------|
| **Corrigé** (confiance élevée) | A + B + C + D du protocole post-fix tous propres ; reproduction confirmée avant ET après ; cause racine identifiée |
| **Corrigé** (confiance moyenne) | A + au moins 1 voisin ; pas de régression évidente ; cause racine plausible |
| **Corrigé à confirmer** | A passé ; voisins limités, ou contrainte environnementale ; lister ce qui n'a PAS été vérifié |
| **Correctif appliqué, validation incomplète** | équivalent ; choisir une formulation par audit |
| **Partiel** | Atténue le symptôme mais ne traite pas la cause racine |

Sans browser disponible → confiance maximum = **moyenne**, jamais **élevée**.

Cette discipline élimine la formule fausse "j'ai corrigé X" alors qu'on n'a fait que modifier le code.

---

## Inflation et déflation

### Anti-inflation
- Pas de section "améliorations possibles" sauf si demandée
- Pas de "best practices non respectées" sans impact démontré
- Pas de copie générique de checklists tierces (OWASP, WCAG) sans contrôle ciblé
- Si un finding ne mérite pas d'action, il n'a pas sa place dans le rapport

### Anti-déflation
- Ne pas adoucir un fail-open auth pour faire joli
- Ne pas reclasser une illusion de fonctionnalité critique en "à améliorer"
- Ne pas masquer une cause racine derrière un symptôme

L'audit doit être **sévère mais juste**. Sévère sur les vrais problèmes, juste sur l'absence de problème.

---

## Style de communication

- "Suspect, non confirmé" est honnête, à préférer à "probablement cassé"
- "Validation incomplète" est honnête, à préférer à "fixed"
- "Aucun pattern à risque détecté lors du scan ciblé" est honnête, à préférer à "sécurité OK"
- "Non testé dans cette session" est honnête, à préférer à un silence

L'utilisateur préfère savoir ce qui n'a pas été couvert plutôt que de découvrir le trou plus tard.
