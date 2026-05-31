A skill that makes **lines of code (SLOC)** a true north star for *actually* simplifying your codebase - preventing your agent from gaming the metric. Aggressively de-bloat your AI generated codebase.

It provides your agent with a **preflight checklist** (to establish baselines and safety nets before deleting anything), an **honest reduction order**, strict **anti-gaming rules** with a **self-audit**, and well-defined **stop conditions** so the process escalates rather than spins. It's language-agnostic, with a Flutter reference and a no-tooling fallback included.

Tested with Claude Code's `/goal` mode.

## Why

A line-count target is a powerful forcing function, but it's easily gamed. The lesson behind this skill:

> Left alone, an agent optimizes the *metric* the cheap way—trimming comments, packing lines, reformatting—and calls it progress. The number drops, but the system doesn't improve.

So this skill is based on two fundamental rules: **ingenuity** (find actual structural wins) and **honesty** (never improve the proxy at the system's expense; surface hard truths). A self-audit requires the agent to classify its own cuts as *structural* vs *cheap* and calls itself out when cheap wins dominate.

## How it worked for us

Based on a real run with a ~20,000-line Flutter app:

|                    | Before |               After |
|--------------------|-------:|--------------------:|
| `sloc --app` total | 19,772 | **13,509** (−31.7%) |
| app code (`lib/`)  | 15,859 |               9,924 |
| tests              |  green |       **335 green** |

All features preserved, analyzer clean, verified on both Android emulator *and* Linux desktop build, with 2 latent bugs fixed along the way.

What worked: deleting dead code and a no-op placeholder subsystem, relocating the debug harness out of shipping code, eliminating a redundant state layer, clean-room rewrites against tests, and delegating custom logging to a library. What *looked* like progress but wasn't: trimming comments (the easiest lever, which undermines the spirit of the task), and "deep-module" reshuffles (solid design, but line count stays ~neutral). The honest part: generated/native scaffolding sets a floor—when the target is below that, the rest is a product decision, not an engineering one. The skill makes the agent surface that math up front and escalate if needed.

## Install

From your project root (or add `-g` for a global install):

```bash
npx skills add maxim-saplin/goal-sloc
```

**Manual alternative:** copy everything under [`skills/goal-sloc/`](skills/goal-sloc/) (`SKILL.md` + `references/`) into your agent’s skills folder—e.g. `.agents/skills/goal-sloc` (works across many harnesses) or  `.claude/skills/goal-sloc` (Claude Code).

## Repository layout

```
README.md
LICENSE
.gitattributes

skills/
  goal-sloc/
    SKILL.md
    references/
      preflight-checklist.md      # Preflight + during-work loop
      minimal-tools.md            # SLOC convention + paste-ready dead-code scripts
      flutter-sloc-reference.md   # Flutter/Dart levers + worked example
```

## Use

Just ask, in plain language:

- "Reduce bloat in this repo; keep all features and tests green."
- "Get SLOC under 15k by simplifying—no gaming the metric—and tell me when only feature cuts are left."

It runs preflight, applies the reduction order with verification after each change, self-audits for structural vs. cheap cuts, and stops (escalating honestly) when the well runs dry.

> **Biggest risk:** No feedback loop. Make sure the agent can run the tests and actually exercise the app—unit tests prove contracts, not "it works."
