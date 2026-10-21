---
name: ship-implement-plan
description: Execute an implementation plan phase by phase, with verification between phases and pauses for manual testing. Consumes the plan from /ship-create-plan, or runs as Ship pipeline stage 5 (the plan from /ship-create-plan).
triggers:
  - /ship-implement-plan
model: opus
---

# Implement Plan

Triggered by: `/ship-implement-plan [plan-file]`
Example: `/ship-implement-plan 2026-04-14-SP-42-plan.md`

Execute an approved implementation plan phase by phase. The plan was produced by `/ship-create-plan` and already incorporates the design decisions and the delivery shape (phases, PR grouping, release boundaries). Your job is to follow the plan's intent, verify as you go, and pause for human checks between phases.

## Inputs

A plan file (output from `/ship-create-plan`). Read it fully. Also read the design discussion referenced in the plan header - it contains the patterns to follow.

If no plan file is provided, ask for one.

## Process

### Step 1: Orient

- Read the plan fully
- Read the design discussion referenced in the plan (for patterns and decisions)
- Check for existing checkmarks (`- [x]`) - if resuming, pick up from the first unchecked item
- Read all files mentioned in the first phase before writing any code

### Step 2: Implement phase by phase

For each phase:

1. Read all files involved in the phase before making changes
2. Build the smallest thing that satisfies the phase (see Restraint below)
3. For logic worth testing, write the failing test first, then implement to make it pass
4. Follow the patterns from the design discussion - don't invent new patterns
5. Update checkboxes in the plan file as you complete items

Code comments describe the code's own intent. Only comment when it is not obvious from the code. Never explain why the change is being made. Never reference plan artefacts: no phase names or numbers, ticket numbers, Ship folders/manifests, plan or design file names, or `.md` files. Those live in the plan, not the source.

### Step 3: Verify each phase

After completing a phase, run the automated verification listed in the plan (tests, types, lint). Fix issues before proceeding.

Then git commit all changes from the phase (excluding the plan file itself). Use a concise commit message describing the phase intent.

Then pause using the `AskUserQuestion` tool with:
- A summary of what was completed and what automated checks passed
- The manual verification steps from the plan listed as the question
- Options: "Ready - proceed to Phase [N+1]" and "Issues found - hold"

Do not proceed until the user confirms via the tool response. Do not check off manual verification items yourself.

If instructed to run multiple phases consecutively, skip the pause and commit until the last phase.

### Step 4: Handle mismatches

If the code doesn't match what the plan expects, stop:

```
Mismatch in Phase [N]:
Expected: [what the plan says]
Found: [actual situation]
Why this matters: [impact on the plan]

Options:
1. [Adapt the current phase to work with what exists]
2. [Flag that the upstream design/structure may need revisiting]
```

Do not silently work around mismatches. The plan was built on specific assumptions from the design discussion - if those assumptions are wrong, the human needs to know.

### Step 5: Review before committing

The last step of the build, before the changes go to git.

First, audit comments. Launch a fresh `general-purpose` subagent (`model: sonnet`) with only the run's diff, not the plan, ticket, or design. Give it these rules verbatim: "No references to tickets or phases. Only comment when it is not obvious from the code. Never explain why the change is being made." It removes every added comment that fails any rule, edits nothing else, and lists what it removed. Amend the removals into the relevant commits or add a follow-up commit.

Then invoke the `ship-antagonistic-review` skill with ARTIFACT = the run's diff, LENSES = `design` plus any risk lens the diff touches (`security`, `data`, `reliability`), CONTEXT = the plan and the design discussion, and the pertinent CLAUDE.md rules verbatim.

Incorporate the valid findings; dismiss the rest with a one-line reason. Then commit.

### Step 6: Close out the Ship stage (only if applicable)

This skill is Ship pipeline stage 5. If the plan file lives inside a `*-ship/` folder that contains a `manifest.md`, then after ALL phases are complete, verified, and reviewed, mark the `Implement plan` stage `done` in that `manifest.md` (see `../ship/manifest-format.md` from this skill's base directory). This is the ONLY manifest write this skill makes, and it is single-threaded (implementation runs in one thread).

For standalone runs (no `*-ship/` manifest), skip this step entirely - there is nothing to update.

## Restraint

The plan says WHAT to build. It rarely dictates every line of HOW. In that latitude, take the lazy path that works: laziness here means less code to own, never less understanding. Trace the flow the change touches end to end first, then climb this ladder and stop at the first rung that holds:

1. **Does this need to exist at all?** If the phase intent is already met, or the piece is speculative "for later", skip it and say so in one line.
2. **Already in the codebase?** Reuse the helper, util, type, or pattern that already lives here before writing new code.
3. **Stdlib or an installed dependency does it?** Use it. Never add a dependency the plan didn't call for without flagging it first.
4. **Can it be one line?** Then one line. Only then write the minimum code that works.

**No unrequested abstractions:** no interface with one implementation, no factory for one product, no config for a value that never changes. Generalise on the third occurrence of a pattern, not the second. Deletion over addition, boring over clever, fewest files, shortest working diff.

**Keep cyclomatic complexity low.** Guard clauses with early exits at the top, happy path at the bottom; prioritise `if` and avoid `else`; avoid nested conditionals and ternary overuse; co-locate errors and side effects with their trigger. Fewer branches is fewer tests, fewer bugs, and less to abstract later.

**Tests earn their place by pinning behaviour, not implementation. YAGNI applies to tests too.** Follow the plan's testing strategy; within it:
- Test the things with real value: contracts, public functions, and user flows - the observable behaviour a caller depends on. These survive refactors and catch actual breakage.
- Don't write tests that just restate the implementation (asserting a private helper was called, mirroring the code's branches, snapshotting internals). They solidify the code as-written, right or wrong, and break on every refactor without catching a single bug.
- Trivial code (a pass-through, a one-line getter, plain wiring) gets no test. No per-function suites, fixtures, or mocks the plan didn't ask for.
- If a test would pass no matter how the behaviour broke, or fail whenever the code is refactored without changing behaviour, delete it. Don't test the framework, the language, or the mock.
- When you do write tests, build test data through factories and keep tests isolated (no shared mutable state, dynamic values pinned at the test boundary).

**Never be lazy about:** input validation at trust boundaries, error handling that prevents data loss, security, accessibility, or anything the plan explicitly requires. Note deliberate corner-cuts (ceiling and upgrade path) in the phase summary, not in code comments.

Restraint applies to the latitude the plan leaves. Where the plan is explicit, follow it; if what it prescribes looks over-built, note the leaner alternative in the phase summary rather than silently swapping it in.

## Rules

- **Restraint by default.** Reuse before writing, minimum code that works, no unrequested abstractions (see Restraint).
- **Flat over nested.** Guard clauses and early exits, happy path last, minimise branching (see Restraint).
- **Test behaviour, not implementation.** Test-first for contracts, public functions, and user flows; no tests that just mirror the code or pin trivial wiring (see Restraint).
- **Follow the plan's intent, not just its letter.** If the codebase has shifted since the plan was written, adapt intelligently but communicate what changed.
- **No research or exploration.** That work is done. If you're missing information, the upstream artifacts are incomplete - flag it.
- **No new patterns.** Use the patterns from the design discussion. If they don't fit, that's a mismatch to surface, not a judgment call to make alone.
- **Minimal comments.** Only when not obvious from the code; never why the change was made; no phases, ticket numbers, Ship folders/manifests, plan/design file names, or `.md` references. Audited by a subagent in Step 5.
- **One phase at a time.** Complete and verify before moving on.
- **Review before committing.** Run `ship-antagonistic-review` on the run's diff, then incorporate the valid findings (see Step 5).
- **Stay under 55 instructions total.**
- **Do not git commit the plan file itself.**
