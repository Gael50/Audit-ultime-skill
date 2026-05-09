# Invariants & Root Cause — Audit Ultime

Comment penser un bug : invariants métier, hypothèses, contre-épreuves, trust boundaries, matrice de vérité. Le but : remonter à la cause racine, pas s'arrêter au symptôme.

Pour les **signatures concrètes** de bugs (recettes Grep, code patterns), voir `detection-patterns.md`.

---

## Invariants métier

Un invariant = une propriété qui doit être vraie à tout moment, par contrat avec l'utilisateur. Identifier les invariants à partir du profil projet (step 2) puis les utiliser pour tester la cohérence.

### Exemples

- Un stock ne doit pas apparaître dans le mauvais entrepôt
- Un onglet masqué ne doit pas redevenir visible après refresh sans raison
- Un renommage sauvegardé doit être relu
- Une suppression ne doit pas réapparaître au reload
- Une vue non-active ne doit pas consommer de ressources si le système promet du lazy loading
- Un bouton icône doit rester accessible
- Un compteur doit correspondre aux données réellement visibles
- Un settings sauvegardé doit être réappliqué à la session suivante
- Un utilisateur ne doit pas voir les données d'un autre

### Méthode

1. Lister 3-5 invariants critiques pour le projet
2. Pour chaque invariant : concevoir 1 scénario navigateur qui peut le violer
3. Exécuter le scénario
4. Si violation → bug confirmé, monter en cause racine via lecture code

Une violation d'invariant compte plus qu'un écart UI cosmétique. Un compteur faux = un mensonge à l'utilisateur.

---

## Hypothèses + contre-épreuves

Pour chaque zone auditée, formuler des hypothèses actives, puis tenter de les invalider.

```
H : "Cette feature semble exister mais pourrait ne pas être réellement branchée."
Contre-épreuve : cliquer dans tous les contextes possibles, observer la mutation réelle.
Résultat : la contre-épreuve échoue → hypothèse résiste → feature semble fonctionnelle.
```

```
H : "Ce setting persiste après reload."
Contre-épreuve : changer le setting, hard reload, observer.
Résultat : la contre-épreuve réussit → hypothèse invalidée → bug confirmé.
```

Hypothèses canoniques à formuler par défaut :

- Cette UI semble exister, mais peut ne pas être branchée
- Cet état UI peut ne pas être persisté
- Ce filtre peut être seulement visuel
- Cette vue peut charger des données inutiles
- Cette config peut être enregistrée mais jamais relue
- Cette logique peut dépendre d'un état global non fiable
- Ce module peut casser dans un ordre d'utilisation différent
- Ce comportement peut diverger entre refresh, navigation interne et retour arrière
- Ce correctif peut masquer un symptôme sans corriger la cause

**Ne pas s'arrêter à la première explication plausible.** Si une lecture suggère que le bug est dans le module A, vérifier aussi si B injecte un patch concurrent.

---

## Trust boundaries

Beaucoup de bugs viennent d'une mauvaise frontière de responsabilité entre couches. Pour tout problème important, vérifier la frontière concernée :

| Trust boundary | Symptôme typique |
|----------------|------------------|
| Front ↔ Backend | Schéma drift (front attend `name`, back retourne `title`) ; UI affiche `undefined` |
| UI ↔ État global | Action UI qui mute un store inattendu, ou état muté hors UI sans rerender |
| Config ↔ Runtime | Variable de config sauvegardée mais lue par un fallback hardcodé |
| Stockage local ↔ Rendu | Valeur écrite mais jamais lue ; ou lue avec default qui masque la valeur |
| Route active ↔ Vue affichée | URL change mais composant ne réagit pas (ou inverse) |
| Module parent ↔ Enfant | Props non passées ; état partagé via globale au lieu de prop |
| Utilisateur ↔ Permission | UI affichée alors que l'API rejette |
| Entrepôt sélectionné ↔ Données affichées | Filtres désynchronisés du store actif |
| Script injecté ↔ Code source | Monkey-patch concurrent du code de base |

Pour chaque bug : nommer la trust boundary impliquée. Aide à formuler la cause racine et à choisir la bonne classe de fix.

---

## Matrice de vérité par feature

Pour chaque fonctionnalité importante auditée, attribuer un statut :

| Statut | Symbole | Signification |
|--------|---------|---------------|
| Réellement fonctionnelle | ✅ | Prouvée par scénarios multiples |
| Fonctionnelle mais fragile | ⚠️ | Marche en happy path, casse au bord |
| Partiellement branchée | 🔶 | UI présente, logique incomplète |
| Décorative / illusion | ❌ | UI prétend que la feature existe, la réalité dit non |
| Non reproductible | ❓ | Impossible de confirmer dans un sens ou l'autre |
| Cassée | 💥 | Confirmée non fonctionnelle |
| Corrigée puis validée | 🛠 | Fixée pendant cet audit + revalidation passée |
| Corrigée mais à surveiller | 👀 | Fixée, régressions adjacentes possibles |

Être impitoyable sur les **illusions**. Une UI qui ment à l'utilisateur est un problème majeur, même si "ça a l'air bien".

---

## Symptôme vs cause racine vs dette

Trois niveaux à distinguer dans le rapport :

- **Symptôme** : ce que l'utilisateur observe
- **Cause racine** : pourquoi ça se produit (file:line)
- **Dette structurelle** : ce qui rend ce type de bug probable dans l'avenir

Exemple :
- Symptôme : "L'onglet actif est perdu au refresh"
- Cause racine : `localStorage.setItem('tab', x)` existe à l'init, `getItem('tab')` n'existe nulle part
- Dette : la persistance est codée à 4 endroits différents avec des conventions de clé qui drift

Le fix court vise la cause racine. La section "Dette technique" vise le pattern.

---

## Audit différentiel obligatoire

Comparer activement les comportements pour faire émerger les divergences :

- Cached vs cold reload
- Onglet actif vs inactif
- Vue visible vs cachée
- Desktop vs mobile (375px)
- Empty state vs populated
- Avant save vs après reload
- Utilisateur configuré vs default
- Entrepôt A vs entrepôt B
- Avec filtre vs sans
- Mode normal vs minimal
- Route directe vs navigation depuis l'app
- Sauvegarde avant reload vs après reload

Toute divergence entre une paire = bug potentiel à investiguer.

---

## Défauts systémiques à chercher activement

Patterns transverses qui produisent les bugs les plus difficiles à diagnostiquer :

- Initialisation dans le mauvais ordre
- Double source de vérité
- État UI et données non synchronisés
- Persistance écrite mais jamais relue
- Config lue une fois mais jamais réappliquée
- Filtre affiché mais non fonctionnel
- Logique métier codée dans des labels UI
- Comportement dépendant d'un clic indirect non garanti
- Code injecté après coup qui concurrence le code source
- Monkey-patch fragile
- Variable closure-scope jamais exportée
- Listeners dupliqués
- DOM rendu puis re-rendu inutilement
- Vues cachées mais toujours actives
- Fallback silencieux qui masque les erreurs
- Valeurs par défaut qui écrasent des données réelles
- Config locale qui change le comportement sans visibilité

Pour chaque défaut systémique trouvé : prouver via lecture code + le mentionner dans la section "Dette technique" du rapport, même si un fix court traite le symptôme.

---

## Protocole de reproduction robuste

Chaque bug confirmé inclut un protocole standardisé :

```
- Contexte initial: <état du système au démarrage>
- Préconditions: <données, settings, user, viewport>
- Actions exactes:
  1. <action>
  2. <action>
- Résultat observé: <ce qui se produit>
- Résultat attendu: <ce qui devrait se produire>
- Fréquence: <toujours / 3 sur 5 / une fois sur 10>
- Variantes testées: <liste>
- Ce qui change si on refresh: <comportement différent ?>
- Ce qui change si on change de vue / filtre / entrepôt: <…>
- Ce qui change en mobile vs desktop si pertinent: <…>
```

Si un bug ne peut pas être reproduit de manière stable → l'indiquer explicitement (Reproductibilité: Non confirmé) + le déclasser à **Confiance: Très probable** maximum.

---

## Cause racine — checklist

Pour tout problème significatif, considérer si la cause vient :

- [ ] D'une erreur de lecture du code par moi
- [ ] D'une mauvaise compréhension de la frontière de responsabilité
- [ ] D'une mauvaise interprétation de l'intention produit
- [ ] D'un couplage masqué entre deux couches
- [ ] D'un patch local contredit par le code source
- [ ] D'un script injecté trop tard / trop tôt
- [ ] D'un sélecteur fragile
- [ ] D'un lazy loading mal configuré
- [ ] D'un ordre d'init incorrect
- [ ] D'un problème de portée (closure)
- [ ] De données mal normalisées
- [ ] D'un backend désaligné avec le front

Cocher mentalement avant de figer la cause racine. Une cause racine identifiée trop vite = soit fausse, soit incomplète.

---

## Génération de tests jetables

Quand pas de suite de tests : la skill génère mentalement des tests jetables ciblés sur :

- Les hypothèses fragiles à invalider
- Les bords (empty, max, négatif, unicode, espaces)
- Les changements récents (focus sur le diff si dispo)
- Les invariants métier identifiés
- Les zones à risque pour la revalidation post-fix

Ces tests servent **immédiatement** dans l'audit. Ne pas attendre une grosse suite de tests automatisés.

Après audit : proposer à l'utilisateur de promouvoir les tests les plus rentables en CI.
