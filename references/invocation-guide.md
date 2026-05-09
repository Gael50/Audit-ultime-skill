# Invocation Guide — Audit Ultime

Comment déclencher la skill, exemples de prompts, variantes de mode (full / compact), et règles d'escalade.

---

## Trigger phrases

Activer sur tout message contenant :

- `audit` / `audite`
- `audit complet` / `audit ultime` / `audit global` / `audit local` / `audit live`
- `audite ce projet` / `audite cette app` / `audite ce site` / `audite cette webapp`
- `fais un audit`
- `audit du site` / `audit webapp` / `site audit` / `webapp audit` / `project audit`
- `audit anti-régression` / `audit fiabilité`
- `fais un audit puis corrige si c'est safe`

Ou toute variante française/anglaise dont l'intention est *"investigate ce projet pour bugs, illusions, causes racines, et livre un rapport prouvé"*.

---

## Modes d'invocation

### Full mode (défaut)
Trigger : "audit", "audit ultime", "audit complet", "audite ce projet"
Comportement : 12 étapes du workflow, battery complète Playwright, jusqu'à 3 fixes safe, rapport complet.

### Compact mode
Trigger : "audit rapide", "audit léger", "audit compact", "quick audit", "audit express"
Comportement : 8-12 minutes total, hard cap. Mêmes règles anti-bullshit, périmètre réduit.

Voir section "Mode compact" plus bas.

### Live mode
Trigger : "audit live", "audit avec navigateur", "audit Playwright", "audit visible"
Comportement : forcer Playwright en mode `--headed` si dispo. Annoncer chaque scénario avant exécution. Privilégier MCP Playwright > Browser MCP > local.

### Audit + repair mode
Trigger : "audit puis corrige", "audite et fix", "audit + repair safe"
Comportement : full audit + apply jusqu'à 3 fixes classifiés SAFE / SAFE+REGRESSION (cf. `repair-rules.md`). Stop avant tout fix MEDIUM ou HIGH-RISK sans confirmation utilisateur.

### Anti-regression mode
Trigger : "audit anti-régression", "audit fiabilité max", "audit non-régression"
Comportement : full audit + accent sur revalidation post-fix (protocole A/B/C/D), confiance honnête (élevée / moyenne / partielle), bloc "Risques résiduels avant merge" obligatoire et détaillé.

---

## Exemples de prompts

```
audit
```
→ Full mode sur le projet courant.

```
audit ultime
```
→ Full mode, déclenchement explicite.

```
audite ce projet
```
→ Full mode, ton conversationnel. Identique.

```
audit rapide
```
→ Compact mode, 8-12 min.

```
audit avec Playwright en mobile
```
→ Full mode, viewport mobile prioritaire (375×812), recettes 9 et 12 obligatoires.

```
fais un audit puis corrige si c'est safe
```
→ Audit + repair mode. Apply SAFE seulement, jamais MEDIUM/HIGH sans accord.

```
audit anti-régression sur le dernier commit
```
→ Anti-regression mode + focus sur le diff git récent.

```
audite cette webapp et donne-moi un rapport priorisé
```
→ Full mode standard.

```
audit local — l'app tourne sur localhost:3000
```
→ Full mode, BASE_URL fournie (skip step 3 launch).

```
audit fiabilité maximale
```
→ Anti-regression mode, ton senior, preuve obligatoire pour chaque finding, confiance graduée.

---

## Mode compact — détails

Variante légère pour vérification rapide. Hard cap **8-12 min**.

### Ce qui change

| Step | Full | Compact |
|------|------|---------|
| 1 — Cartographie | Tous configs + entry points + storage | CLAUDE.md + package.json + top-level only |
| 2 — Profil | 7 champs | TYPE / STACK / LAUNCH only |
| 3 — Bring-up | Auto-launch | Réutiliser serveur existant ; sinon code-only |
| 4 — Browser | Recettes 1, 3, 4, 5, 6, 7, 9, 10, 11, 12 | Recettes 1, 3, 4 only |
| 5 — Code | 25 patterns ciblés | 5 patterns highest-yield (P1, P3, P4, P7, P10 du `detection-patterns.md`) |
| 6 — Performance | DOM + network + render | DOM only |
| 7 — A11y | 6 contrôles | Skip sauf flag step 4 |
| 8 — Sécurité | Scan 5 patterns | Skip sauf red flag step 5 |
| 9 — Invariants | 3-5 invariants | 1 invariant le plus critique |
| 10 — Auto-fix | 3 fixes | 1 fix max, SAFE obligatoire |
| 11 — Revalidation | A + B + C + D | A only |
| 12 — Rapport | Template complet | Rapport compact (ci-dessous) |

### Rapport compact

```markdown
# Audit Compact — <project> — <date>

## Verdict en une phrase
<une phrase : état du projet + problème le plus critique>

## Top 3 problèmes confirmés
1. <bug + priorité + 1-line cause + 1-line fix>
2. ...
3. ...

## Quick wins (≤30 min de fix)
- ...

## Ce qui fonctionne (sanity check)
- <2-3 bullets confirmant les flux critiques>

## Suspicions à vérifier plus tard
- ...

## Méthode
- <quels scénarios ont tourné>
- <quels patterns ont été checkés>
- <ce qui n'a PAS été couvert>

## Auto-doubt
- <ce qui est prouvé>
- <ce qui est inféré>
- <blind spots>
```

Longueur totale : 30-50 lignes max. Pas de tableaux sauf nécessaire.

### Escalade obligatoire en compact

Si pendant un compact audit l'un de ces signaux apparaît :
- Bug **Critique**
- Anomalie auth
- Incohérence de persistance (saved mais jamais lu)
- Illusion sur une feature critique user-facing

→ stopper le compact, dire à l'utilisateur : *"Trouvé <X>. Recommande d'escalader vers audit complet."* Attendre confirmation. **Ne pas étendre le scope silencieusement.**

### Invariants en compact

Ce qui est tronqué : le **scope**.
Ce qui n'est jamais tronqué : la **rigueur**.

- Les scénarios browser tournent quand même avec screenshots (compact ≠ aveugle)
- Les findings utilisent quand même le template par finding (compact ≠ vague)
- Le scoring est quand même appliqué (compact ≠ non priorisé)
- L'auto-doubt est **obligatoire** (compact ≠ overconfiant)

---

## Pre-flight check à chaque invocation

Avant de lancer le workflow, la skill annonce :

1. Quel mode (full / compact / live / repair / anti-regression)
2. Quels outils détectés (browser MCP / local Playwright / aucun)
3. Quel viewport (desktop / mobile / les deux)
4. Source de la BASE_URL (localhost:port / déployée / static)
5. Time-box estimé

Si une donnée manque → demander une fois (pas en boucle). Si l'utilisateur ne répond pas → choisir le défaut le plus prudent et flagger dans auto-doubt.

---

## Garde-fous d'invocation

### Ne pas auto-lancer si
- Un audit est déjà en cours dans la même conversation (vérifier avec TaskList si dispo)
- Le projet n'est pas dans le working directory courant et la skill ne sait pas où regarder
- L'utilisateur a écrit "audit" dans un contexte où ce n'est pas une demande (ex. "j'ai relu l'audit d'hier")

### Toujours confirmer
- Avant d'appliquer un fix MEDIUM ou HIGH-RISK
- Avant d'installer une dépendance (`npm install playwright`)
- Avant de lancer un serveur sur un port qui pourrait être déjà occupé par un service tiers
- Avant de faire un git commit, sauf si l'utilisateur l'a explicitement autorisé en amont

---

## Composition avec d'autres skills

L'audit-ultime peut être chaîné avec :

- **`webapp-testing`** — pour des tests Playwright simples sans le full audit
- **`security-review`** — pour aller plus loin sur la sécurité après que l'audit ait flagué une zone
- **`fix`** — pour formater / lint avant commit après les fixes
- **`mode-fiabilite-max`** (si la skill séparée existe) — overlay reliability discipline ; en pratique déjà intégré dans `repair-rules.md` et `anti-false-positives.md`

L'audit-ultime ne lance pas les autres skills automatiquement — l'utilisateur les invoque ensuite si nécessaire.
