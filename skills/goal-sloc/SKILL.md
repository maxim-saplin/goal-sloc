---
name: goal-sloc
description: Playbook for using lines-of-code (SLOC) as a north-star metric to genuinely simplify and improve an engineering solution — without gaming the number. Use when asked to "cut SLOC", "reduce/simplify the codebase", remove bloat/complexity/tech-debt, delete dead code or duplication, or hit a line-count target. Covers preflight (baseline + feedback loops + tooling), an honest reduction order, a self-audit against gaming, stop conditions, and a Flutter reference.
---
# goal-sloc — SLOC as a north-star, honestly

**SLOC is a proxy, not the goal.** The goal is a *simpler, better-engineered, still-correct* system. SLOC is useful because it forces you to find dead weight and accidental complexity and to make real decisions. It is dangerous because it is trivially gameable, and because the cheapest ways to move it (deleting comments, packing/formatting lines, extracting helpers) often do **not** improve the system. Every line you remove is a claim you must defend: *"the system is now better-or-equal, and still works."*

Two non-negotiables: **ingenuity** (find the real structural wins) and **honesty** (never optimize the proxy at the system's expense; surface hard truths instead of hiding them).

### Self-audit (the load-bearing rule)
The dominant failure mode: *left alone, you will optimize the metric the cheap way and call it progress.* Defend against your own instinct:
- Every few milestones, **classify your reductions: % from structural work** (dead/placeholder removal, relocation, de-duplication, re-architecture) **vs. % from cheap levers** (comment trimming, formatting, whitespace, helper extraction).
- **If cheap levers dominate, you are gaming yourself — stop and find structural work**, or report that the structural well is dry (which is itself the honest finding).
- Put this split in your milestone reports to the human; don't make them discover it.

---

## 0. Preflight — DO THIS BEFORE CUTTING ANYTHING
Skipping preflight is the #1 cause of broken, untrustworthy "simplification." Work the copy-paste **`references/preflight-checklist.md`**; don't start cutting until each item is answered.

**0a — Pin the target & understand the metric.**
- Confirm the *current* target: what number, **measured by exactly what tool/command**, and **has the bar moved?** SLOC goals are often social/moving, not in config — re-confirm whenever the human raises it; don't trust a stale goal artifact.
- Read the measuring tool. Which paths/files does it count (tracked-only? generated? binary? tests?)? If *you* must define the metric, default to **non-blank, non-comment-only lines** and state whether doc-comments count, so results are comparable.
- Record the baseline total + per-area breakdown.

**0b — Establish feedback loops (the critical precondition).** You cannot safely reduce code you cannot verify. Rank your safety nets and **build the missing ones first**:
1. **Tests** green at baseline, with real coverage on what you'll touch. **Sparse tests = the dominant risk** — augment coverage there first, or proceed with extreme caution + heavy runtime checks.
2. **Static analysis / type check / lint** clean. *No analyzer? The minimum bar is a parse/compile check + tests.* Re-run after every change.
3. **Run and EXERCISE the real system end-to-end** — by whatever fits its shape: an app driver / UI automation, HTTP calls, CLI invocations, integration tests, or (for a library) its public API via consumers/examples. **The most common, most dangerous gap is having no runtime feedback loop** — unit tests prove contracts, not "it actually works." If you can't exercise it, that's a top risk: build a driver or flag it loudly. *Actually run it during the work.*

**0c — Guard rollback.** Work on a branch (never main). **Commit per verified milestone** so any change is cheaply revertible — aggressive deletion/rewrite is exactly when you need it.

**0d — Map the code with real tools (not grep).** Get a **dead-code/unused-export report** and a **dependency graph + complexity hotspots** from a *semantic* tool. **grep is not code intelligence** — it misses relative imports, re-exports, tear-offs, dynamic/string dispatch → false "dead code." Use grep only to *confirm* a tool's hit. Tooling fallback ladder when nothing is installed and you can't install: language server → a minimal AST script (**`references/minimal-tools.md`** has paste-ready snippets) → grep-to-confirm-only.

**0e — Categorize & compute the floor.** Split counted code into **shipping**, **test/dev scaffolding**, and **generated/platform/vendored** (the "floor"). Compute the irreducible floor (generated configs, binary assets counted as lines, third-party) and do the arithmetic: is the target even reachable above `floor + genuine feature minimum`? If not, say so now — the remaining gap is a *product/scope* decision, not engineering.

---

## 1. The honest reduction order
Legitimacy/value order — **but risk rises as you go down**: items 1–2 are safe; 6–8 are high-risk and require strong feedback loops (§0b). Verify after each (§4).

1. **Delete dead code** — files/exports/functions with *no* references (semantic-tool-verified, not grep). Pure win.
2. **Remove placeholder / no-op subsystems** — fully plumbed features that do nothing (a toggle hardcoded "unavailable", a method that's a logged no-op). Dead weight as features. Pure win.
3. **Relocate misplaced code to its correct home** — test/dev/automation scaffolding in shipping source → `test/` or a `dev/` area behind a thin seam. A real structure fix the (unchanged) counter rewards. (Moving a file ≠ editing the counter — §2.)
4. **De-duplicate genuine duplication** — repeated blocks → one parameterized helper; N near-identical files → one. (Often ~LOC-neutral; do it for clarity.)
5. **Comment/verbosity hygiene** — collapse restate-the-code comments and AI-left noise. *Legitimate, but the spirit-trap:* the cheapest lever, satisfying drops, zero design gain. It may **ride along for free on files you're already rewriting**; the trap is doing it *standalone as your strategy*. If it's most of your "progress," see the self-audit.
6. **Tests-as-spec clean-room rewrites** — rewrite the biggest files from scratch against their tests, discarding accumulated cruft. **Only safe when the tests are a faithful behavioral spec** — otherwise you rewrite bugs into features. (A modest density gain is typical on already-trimmed code; not a target.)
7. **Architectural simplification** — collapse redundant layers, kill antipatterns, deepen modules. **Mostly SLOC-neutral-to-positive — do it for *quality*, not the number.** Beware the traps in §5.
8. **Delegate to libraries** — replacing hand-rolled code with a mature dependency removes it from your counted tree *and* is usually better — but only when the lib is genuinely the better choice, never as a counter-dodge, and weigh the dep cost.

When 1–8 are exhausted and you're still above target, the only lever left is **feature/scope/platform cuts** — a product decision. **Escalate to the human with the floor math; never silently delete features.**

---

## 2. Gaming vs. honesty (read twice)
The line: did **the system actually get simpler/better**, or did you just move the number?

| Move | Verdict | Why |
|---|---|---|
| Edit the measuring tool to exclude paths | ❌ Gaming | Changing the ruler, not the thing. |
| Widen the formatter / pack lines / strip blanks to win | ❌ Gaming | Cosmetic; no real change; harms readability. |
| Delete real features/docs to win | ❌ Gaming (unless dead/placeholder) | Scope cut disguised as cleanup — escalate. |
| Move test/dev code to test/dev dirs | ✅ Legit | Correct structure; counter follows reality. |
| Remove dead/placeholder/duplicate code | ✅ Legit | Genuinely less system. |
| Trim restate/AI-noise comments | ✅ Legit hygiene — never a *strategy* | Easy lever that shades the spirit if over-relied on. |
| Delegate hand-rolled code to a real lib | ✅ Legit (if better engineering) | Less owned code + better. |

**Measurement hazard:** auto-formatters can silently *re-inflate* the very lines you trimmed, making "wins" illusory. Pin one formatting decision; **measure format-normalized deltas** (reformat a copy consistently and diff) to separate real reduction from formatting noise. Don't let sub-agents reformat ad hoc.

**Honesty obligations:** report the floor truthfully; when the target conflicts with constraints (keep all features / platforms / no gaming), surface it with data and let the human choose; never claim "it works" you haven't run; report what you skipped and why; report the structural-vs-cheap split (self-audit).

---

## 3. Stop conditions & escalation
- **Diminishing returns:** when consecutive milestones yield little with rising risk, OR every remaining lever is SLOC-neutral architecture — **stop and report; don't manufacture churn.** (Beware refactor→regret→revert loops.)
- **Scope-creep guard:** don't start a re-architecture you can't *finish and verify* this session. A half-collapsed layer / half-migrated pattern is worse than none.
- **Floor reached:** when only feature/scope/platform cuts remain, escalate the tradeoff to the human with numbers — that's their call.

## 4. Verification cadence
Defined once, in **`references/preflight-checklist.md` → "During-work loop."** In short: static-analysis-or-compile + relevant tests after **every** change; at **milestones** run the full suite and **run + adversarially exercise** the real system on representative targets (spam rapid actions, feed bad input, flip modes — this is where latent and newly-introduced bugs surface); then re-measure and commit.

## 5. Traps & risks
| Risk | Mitigation |
|---|---|
| No/weak tests → refactoring blind | Augment coverage on touched areas first; lean on runtime exercising. |
| No runtime feedback loop | Build/secure end-to-end exercising *before* cutting; actually run it. |
| grep-as-analysis → false dead code | Semantic tool; grep only to confirm. |
| Formatter inflation | Pin one format; measure format-normalized deltas. |
| "Deep module / refactor ≠ fewer lines" | Cohesion work adds scaffolding; do it for design, don't sell it as a SLOC win. |
| **God-object trap** | Collapsing a redundant layer can *over-concentrate* responsibilities into the receiver — you recreate the smell you removed. Extract cohesive collaborators after collapsing. |
| Parallel agents stomp the tree | Run risky/overlapping edits **sequentially**; parallel only on disjoint files; verify globally after. |
| **Treating generated/platform/native as untouchable OR free** | A common default blindspot: generated scaffolding *is* a floor, but real custom native/platform code is reducible like any code. Know which is which; don't dismiss the latter. |
| Diminishing returns mistaken for failure | Recognize when only scope cuts remain; escalate (§3). |

## 6. References
- **`references/preflight-checklist.md`** — copy-paste preflight + the canonical during-work loop.
- **`references/minimal-tools.md`** — SLOC-counting convention + paste-ready dead-code/orphan-import scripts when no semantic tooling is installed.
- **`references/flutter-sloc-reference.md`** — Flutter/Dart specifics (hooks vs StatefulWidget, GlobalKey→controller, provider/ChangeNotifier layering, sealed classes, data-driven UI, the platform floor, dart format, DCM/GitNexus, app-driving via VM-service extensions).
