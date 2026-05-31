# goal-sloc preflight checklist

Run before cutting a single line. Don't proceed past a ❓ you can't answer.

## Metric
- [ ] Read the measuring tool/script. Exactly which paths/files does it count? Tracked-only? Generated/binary/tests included?
- [ ] Recorded baseline total + per-area breakdown.
- [ ] Know the **floor**: generated/platform/vendored/binary lines that are effectively irreducible. (For Flutter: compare against a fresh `flutter create`.)
- [ ] Did the arithmetic: is the target even reachable above the floor + genuine feature minimum? If not, flag it now.

## Feedback loops (build the missing ones FIRST)
- [ ] Test suite runs green at baseline. Coverage on the areas I'll touch is adequate (else augment first).
- [ ] Static analysis / type check clean at baseline.
- [ ] I can **run the real app/service** on a representative target.
- [ ] I can **drive/exercise it** (driver script, VM-service extensions, integration tests, HTTP). If not → top risk; build it or escalate.
- [ ] I have actually run it once and seen it work (not just unit tests).

## Code map (semantic tools, not grep)
- [ ] Dead-code / unused-export / unused-file report from a semantic tool.
- [ ] Dependency graph + fan-in/fan-out + complexity hotspots.
- [ ] Categorized: shipping code vs test/dev scaffolding vs generated/platform.

## Guardrails set
- [ ] Working on a branch (not main); commits are small and verifiable.
- [ ] Formatting decision pinned; won't let ad-hoc reformat inflate/muddy deltas.
- [ ] Anti-gaming rules understood (SKILL §2): no editing the ruler, no cosmetic packing, no silent feature deletion.
- [ ] A place to log milestones (a `goal-sloc.md` retrospective).

## During-work loop (repeat)
1. Make one focused change (prefer the SKILL §1 order: dead → placeholder → relocate → dedup → hygiene → rewrite → architecture).
2. Static analysis + relevant tests green.
3. At milestones: full suite + run & adversarially drive on mobile *and* desktop targets; re-measure; commit.
4. When only feature/scope/platform cuts remain → stop and escalate to the human with the floor math.
