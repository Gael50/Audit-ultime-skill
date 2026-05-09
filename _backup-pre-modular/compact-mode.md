# Compact Mode — Audit Ultime

Lightweight variant for quick checks. Triggers: "audit rapide", "audit léger", "audit compact", "quick audit", "audit express".

**Time budget:** 8–12 minutes total. Hard cap.

**Same anti-bullshit rules apply** — no fake confidence, evidence per finding, no vague conclusions.

---

## What changes vs full mode

| Step | Full mode | Compact mode |
|------|-----------|--------------|
| 1 — Cartography | Read all configs + entry points + storage | Read CLAUDE.md + package.json + top-level dirs only |
| 2 — Profile | Full 7-field block | TYPE / STACK / LAUNCH only |
| 3 — Local bring-up | Auto-launch + wait | Use existing server only; if none, code-only audit |
| 4 — Browser audit | Recipes 1, 3, 4, 5, 6, 7, 9, 10, 11, 12 | Recipes 1, 3, 4 only |
| 5 — Code audit | Targeted reads + 25 patterns | 5 highest-yield patterns: P1, P3, P4, P7, P10 |
| 6 — Performance | DOM size + network + render time | DOM size only |
| 7 — Accessibility | Full 6 checks | Skip unless flagged in step 4 |
| 8 — Security | Targeted 5-pattern scan | Skip unless red flag in step 5 |
| 9 — Data invariants | Define + test 3-5 invariants | Test 1 most-critical invariant |
| 10 — Auto-fix | Up to 3 fixes | Up to 1 fix, must be Safe-classified |
| 11 — Re-validation | Originating + 2 adjacent | Originating scenario only |
| 12 — Report | Full template | Compact report (below) |

---

## Compact report structure

```markdown
# Audit Compact — <project> — <date>

## Verdict en une phrase
<single sentence: state of the project + most critical issue>

## Top 3 problèmes confirmés
1. <bug + priority + 1-line cause + 1-line fix>
2. ...
3. ...

## Quick wins (≤30min de fix)
- ...

## Ce qui fonctionne (sanity check)
- <2-3 bullets confirming critical flows work>

## Suspicions à vérifier plus tard
- ...

## Méthode
- <which scenarios ran>
- <which patterns checked>
- <what was NOT covered>

## Auto-doubt
- <what's proven>
- <what's inferred>
- <blind spots>
```

Total report length: ~30-50 lines max. No tables unless absolutely needed.

---

## When to escalate to full mode

If during compact audit any of these surface:
- A **Critique**-gravity bug
- An auth-related anomaly
- A persistence inconsistency (saved but not read)
- An illusion of a critical user-facing feature

→ Stop compact mode. Tell the user: "Found <X>. Recommend escalating to full audit." Wait for confirmation.

Don't silently expand scope.

---

## Compact-mode invariants

- Browser scenarios still run with screenshots (compact ≠ blind)
- Findings still use the per-finding template (compact ≠ vague)
- Scoring still applied (compact ≠ unprioritized)
- Auto-doubt section is **mandatory** (compact ≠ overconfident)

What's dropped is **scope**, not **rigor**.
