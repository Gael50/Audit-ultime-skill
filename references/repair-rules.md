# Repair Rules — Audit Ultime

Discipline de correction. Intègre le mode fiabilité maximale (règles 53-65) : **d'abord ne pas casser**.

L'objectif n'est pas le nombre de fixes appliqués, mais la stabilité après audit. 3 fixes prouvés > 10 fixes fragiles.

---

## Principe directeur : d'abord ne pas casser

Avant chaque modification, répondre à ces 6 questions. Si une seule réponse est incertaine → réduire le périmètre, proposer un fix plus sûr, ou stopper et classer "à planifier".

1. Le diagnostic est-il assez solide ? (cause racine identifiée à file:line + reproduction)
2. La zone touchée est-elle critique ?
3. Le fix est-il local ou transversal ?
4. Le risque de régression est-il acceptable ?
5. Existe-t-il une version plus simple et plus sûre du fix ?
6. Peut-on tester immédiatement après ?

Le coût d'un fix postponé = 0 régression. Le coût d'un fix mal classé = potentiellement majeur.

---

## Pré-conditions absolues

Aucun fix ne s'applique sans :

1. **Reproduction** — un scénario navigateur ou un cas runtime qui démontre le bug
2. **Compréhension** — la cause racine est identifiée à un endroit précis (file:line)
3. **Estimation du risque** — la classification SAFE/MEDIUM/HIGH a été choisie consciemment
4. **Plan de re-test** — on sait quel(s) scénario(s) rejouer juste après

Si un seul élément manque → ne pas appliquer. Documenter l'idée de fix dans "Suspicions" ou dans la section dédiée du rapport.

---

## Classification des correctifs

Tout fix proposé ou appliqué reçoit une classe avant action.

### SAFE FIX
- Local, ≤ 10 lignes
- Aucune API publique modifiée
- Aucun caller à refactor
- Cause racine ciblée
- Re-test rapide possible
- **Action** : appliquer, puis re-run scénario d'origine

### SAFE FIX + REGRESSION CHECK
- Correction raisonnable mais touche une fonction consommée par ≥1 autre zone
- Une vue, un module, ou un état partagé pourrait être affecté
- **Action** : appliquer, puis re-run scénario d'origine + 1-3 scénarios voisins

### MEDIUM RISK FIX
- Touche une zone plus large (état partagé, contrat de fonction, persistance)
- Isolation possible mais demande une matrice de revalidation complète
- **Action** : appliquer **uniquement avec accord explicite utilisateur**, puis revalidation full

### HIGH RISK / DO NOT AUTO-APPLY
- Architecture change
- Résolution de double source-of-truth
- Casse une API publique ou un contrat
- Migration de schéma / données
- Upgrade de dépendance
- **Action** : **ne pas appliquer**. Documenter le patch proposé dans le rapport ; argumenter le report.

---

## Heuristiques de classification

| Indicateur | Classe |
|------------|--------|
| Fonction privée à 1 caller, change interne | SAFE |
| Fonction sur `window.*` ou exportée | SAFE+REG (chercher les callers d'abord) |
| Touche persistance / migration / auth / argent / data utilisateur | MEDIUM minimum |
| Schema / contrat / feature retirée / dep upgrade | HIGH |
| Suppression d'un fallback existant | MEDIUM minimum |
| Ajout d'une condition stricte qui peut fail-close | MEDIUM (tester l'ancien chemin) |

**Règle d'or** : en cas de doute entre deux classes → choisir la plus risquée. Le coût d'une sur-classification = un check supplémentaire. Le coût d'une sous-classification = une régression.

---

## Mini protocole post-fix obligatoire (A/B/C/D)

Pour SAFE / SAFE+REGRESSION / MEDIUM. Si un seul bloc est sauté, le statut devient `corrigé à confirmer`, jamais `corrigé`.

### A. Test ciblé
- Rejouer le scénario qui exposait le bug
- Confirmer qu'il ne se produit plus

### B. Test voisin
- Identifier 1 à 3 flux proches potentiellement affectés (même module, même état partagé, même call site)
- Les rejouer
- Confirmer qu'ils passent toujours

### C. Test de stabilité
- Refresh / hard reload si pertinent
- Aller-retour navigation si pertinent
- Switch d'onglet / vue / filtre si pertinent

### D. Test de bruit technique
- Console : avant vs après, aucune nouvelle erreur
- Network : aucun nouveau 4xx/5xx
- Aucun symptôme évident nouveau

---

## Anti-effets de bord — checklist

Avant de déclarer un fix "isolé", répondre :

- [ ] Qu'est-ce qui dépend de cette fonction ? *(grep sur le nom)*
- [ ] Qu'est-ce qui lit cette config ? *(grep sur la clé)*
- [ ] Qu'est-ce qui partage cet état ?
- [ ] Quelle autre vue utilise la même logique ?
- [ ] Quel fallback pourrait être impacté ?
- [ ] Quel comportement mobile pourrait diverger ?

Aucun fix n'est isolé sans cette vérification minimale. Pour une fonction publique, le grep des callers est obligatoire avant SAFE.

---

## Régression intelligente après lot de fixes

Après chaque batch (1-3 fixes), une régression ciblée — pas un re-test du projet entier au hasard. Sélectionner les tests selon :

- Fichiers touchés
- Zone fonctionnelle touchée
- Dépendances visibles
- Invariants métier concernés
- Régressions probables historiques

Exemples concrets :
- Onglets modifiés → navigation, refresh, tab actif, tab masqué
- Entrepôt modifié → entrepôt actif, filtres, compteurs, vues stock
- Settings modifiés → save, reload, réapplication config
- UI critique modifiée → desktop + mobile + console

Ne pas re-tester ce qui n'est pas relié au changement. Ne pas sauter ce qui est relié.

---

## Hard rules

- Jamais bundler plusieurs fixes risqués dans un seul commit
- Jamais déclarer "corrigé" sans re-run du scénario
- Jamais supprimer une feature sans accord explicite
- Jamais remplacer un bug par un workaround qui masque le symptôme
- Jamais introduire une seconde source de vérité
- Jamais chaîner > 3 fixes par audit sans accord utilisateur
- Jamais faire de "drive-by cleanup" pendant un fix : la zone modifiée se limite à la cause racine

---

## Comparaison avant/après obligatoire

Pour chaque fix appliqué, le rapport contient :

```
**Avant:**     <symptôme observé, file:line ou step de scénario>
**Correction:** <ce qui a été modifié, file:line + 1 phrase>
**Après:**     <comportement observé après revalidation>
**Vérifications associées:** <flux voisins testés, console/network checkés>
**Confiance:** élevée / moyenne / partielle
```

### Niveaux de confiance après fix

| Confiance | Conditions |
|-----------|-----------|
| **élevée** | A + B + C + D tous propres ; reproduction confirmée avant ET après ; cause racine identifiée à file:line |
| **moyenne** | A + au moins un voisin testé ; pas de régression évidente ; cause racine plausible |
| **partielle** | A testé mais voisins limités, ou contrainte environnementale a empêché la validation complète ; lister explicitement ce qui n'a PAS été vérifié |

**Sans browser réel disponible** → confiance maximale = **moyenne**, jamais **élevée**, quel que soit le niveau de certitude code.

---

## Quand refuser l'auto-fix

Refuser l'application automatique si :

- Cause racine non identifiée (la classer comme suspicion + proposer investigation)
- Le bug est intermittent et la repro instable (ajouter du logging avant de fixer)
- Le fix demande de toucher un schéma ou un contrat
- Le risque de régression est non bornable
- L'utilisateur n'est pas devant pour valider un fix MEDIUM

Dans le rapport, ces cas vont dans **"Correctifs non appliqués car trop risqués"** avec patch proposé + raison du report.

---

## Anti-patterns interdits

Cette skill rejette activement :

- *"Ça devrait marcher maintenant"* sans rejouer le scénario
- Longues listes de findings basse confiance pour gonfler le rapport
- *"J'ai corrigé partout"* après un grep-replace sans vérifier les call sites
- *"Pas de régression visible"* sans nommer ce qui a été retesté
- Fix qui ajoute un workaround masquant le symptôme au lieu de retirer la cause
- Fix qui introduit une seconde source de vérité plutôt que de soigner la première
- 3+ fixes medium-risk dans un commit
- "Fixed" alors que le seul test consistait à exécuter le code défectueux à nouveau

Si la skill se voit dériver vers un de ces patterns → stop + reclassification.

---

## Stop condition par défaut

**3 fixes par audit** maximum sauf demande explicite de l'utilisateur.

Raison : préserver le budget de revue cognitive. Au-delà, l'utilisateur ne peut plus relire correctement et les régressions passent inaperçues. Si plus de 3 fixes nécessaires → livrer 3 fixes prouvés + lister les autres en "Correctifs proposés non appliqués".
