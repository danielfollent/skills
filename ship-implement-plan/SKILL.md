---
name: ship-implement-plan
description: Execute an implementation plan phase by phase, with verification between phases and pauses for manual testing. Consumes the plan from /ship-create-plan, standalone or as Ship pipeline stage 5.
triggers:
  - /ship-implement-plan
model: opus
---

# Implement Plan

Usage: `/ship-implement-plan [plan-file]` (e.g. `2026-04-14-SP-42-plan.md`). If no plan file is given, ask for one.

The plan already encodes the design decisions and delivery shape. Follow its intent, verify as you go, and pause for human checks between phases.

## Process

1. **Orient.** Read the plan and the design discussion it references. If resuming, start at the first unchecked `- [ ]` item.
2. **Implement each phase.** Read every file involved first. Build the smallest thing that satisfies the phase (see Restraint), test-first for logic worth testing, using the design's patterns. Tick plan checkboxes as you go.
3. **Verify each phase.** Run the plan's automated checks and fix failures. Commit the phase (never the plan file) with a concise message. Then pause with `AskUserQuestion`: summarise what was done and which checks passed, list the plan's manual verification steps, and offer "Ready - proceed to Phase [N+1]" / "Issues found - hold". Never tick manual items yourself. If told to run several phases consecutively, skip pauses and commit after the last.
4. **Stop on mismatches.** If the code doesn't match the plan's assumptions, don't work around it silently:
   ```
   Mismatch in Phase [N]:
   Expected: [what the plan says]
   Found: [actual situation]
   Why this matters: [impact]
   Options: 1. Adapt this phase  2. Revisit upstream design/plan
   ```
5. **Review.** After the final phase:
   - Launch a fresh `general-purpose` subagent (`model: sonnet`) given only the run's diff and these rules verbatim: "No references to tickets or phases. Only comment when it is not obvious from the code. Never explain why the change is being made." It removes failing added comments, edits nothing else, and lists the removals.
   - Invoke `ship-antagonistic-review` with ARTIFACT = the diff, LENSES = `design` plus any touched risk lens (`security`, `data`, `reliability`), CONTEXT = the plan, the design discussion, and pertinent CLAUDE.md rules verbatim.
   - Apply valid findings, dismiss the rest with a one-line reason, and commit.
6. **Close out (Ship runs only).** If the plan sits in a `*-ship/` folder with a `manifest.md`, mark the `Implement plan` stage `done` there (format: `../ship/manifest-format.md`). This is the skill's only manifest write. Standalone runs skip this.

## Restraint

Where the plan is explicit, follow it; if it looks over-built, note the leaner option in the phase summary rather than swapping it in. In the latitude it leaves, trace the affected flow end to end, then stop at the first rung that holds:

1. Does it need to exist? Skip speculative or already-met work, and say so.
2. Already in the codebase? Reuse it.
3. Stdlib or an installed dependency? Use it. Flag any new dependency first.
4. Otherwise, the minimum code that works.

- No unrequested abstractions; generalise on the third occurrence, not the second. Shortest working diff, fewest files, boring over clever.
- Keep complexity low: guard clauses first, happy path last, no `else` where an early exit works, no nesting, errors co-located with their trigger.
- Tests pin behaviour (contracts, public functions, user flows), not implementation. No tests for trivial wiring, private helpers, or the framework; delete any test that survives broken behaviour or breaks on a pure refactor. Use factories and keep tests isolated.
- Never cut corners on trust-boundary validation, data-loss prevention, security, accessibility, or explicit plan requirements. Note deliberate corner-cuts in the phase summary.

## Rules

- Comments only where the code isn't self-explanatory. Never explain why the change was made or reference phases, tickets, Ship folders, plan/design files, or `.md` files.
- No research or new patterns. Missing information or ill-fitting patterns are mismatches to surface.
- If the codebase has shifted since planning, adapt to the intent and say what changed.
- One phase at a time, verified before moving on.
