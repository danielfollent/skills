---
name: ship-create-plan
description: Ship pipeline stage 4. Takes design.md + all research from the per-ticket manifest folder, decides the delivery shape (first usable milestone, session-sized vertical slices, PR grouping, release boundaries, phase order) and expands it into a detailed tactical implementation plan, hardened by a design-lens antagonistic review. Writes plan.md into the manifest folder and hands off to /ship-implement-plan.
triggers:
  - /ship-create-plan
model: opus
---

# Ship Create Plan

Triggered by: `/ship-create-plan`

Produce a detailed, actionable implementation plan that an agent can execute phase by phase. The
human reviewed the design. This stage first decides the delivery shape (how the work is cut,
grouped into PRs, and released), then translates it into tactical instructions. Both live in
`plan.md`; the shape is the part a human skims.

Folder + manifest conventions: `manifest-format.md` in the `ship` skill folder (`../ship/manifest-format.md` from this skill's base directory).

## Inputs

Resolve the ticket folder per `manifest-format.md`, then read fully:

1. **design.md** - the settled design (required)
2. **All `research-<repo>.md`** files - the codebase ground truth
3. **work-breakdown.md** - legacy only (runs started under the old six-stage pipeline). When
   present, use it as the starting delivery shape and re-validate it against `design.md`. Never
   require or regenerate it.

If the design or research is missing, ask the user (the upstream stage probably did not run).

## Process

### Step 1: Decide the delivery shape

Before any file-level detail, settle the shape from `design.md`, the repo breadth in the
manifest, and any rollout / migration / feature-flag cues in the design:

- **First usable milestone** - the thinnest end-to-end increment worth demoing or releasing.
  Everything else is ordered after it.
- **Phases** - vertical slices, each a thin working end-to-end increment, never "Phase 1: all
  models, Phase 2: all controllers". Size each to ONE `/ship-implement-plan` run in a fresh
  thread: plan + design + touched files + tests + verification comfortably under ~150k tokens.
  If a phase cannot finish in one session, split it: first into THINNER vertical slices; only
  when no thinner slice exists, into sequential sub-phases (scaffold -> wire up -> integrate)
  verified by automated tests rather than end-to-end behaviour.
- **Order and dependencies** - sequence phases so earlier ones do not force rework of later
  ones. Name real inter-phase dependencies only; do not invent ordering ceremony.
- **PR grouping** - which phases land in which PR (per repo when several are touched). This is
  a separate decision from deployment order.
- **Release boundaries** - which PRs deploy together and in what order. Anything released
  independently of later work must stand alone: safe with later phases absent, flag-gated or
  backward compatible as the design dictates.
- **Verification per phase** - what proves each phase works: end-to-end where the slice allows,
  automated tests otherwise, plus any manual check that gates the next phase or a release.

This is a spectrum, not a binary. One ticket may be one PR with three build-and-verify phases;
an initiative may be several releases each holding several PRs. Choose the lightest shape the
design supports and decide it yourself. Ask the user only when the design leaves a shape
question genuinely open (e.g. whether a slice may ship ahead of its partner).

### Step 2: Gather implementation details

For each phase, spawn parallel sub-agents to find:
- Exact files that need changes (with line numbers)
- Code patterns to follow (from the design's "Patterns to Follow")
- Existing test patterns in the relevant areas
- Import conventions, naming conventions, config patterns

If a phase turns out larger than one session once its files are known, return to Step 1 and
split it.

### Step 3: Write the plan

Write `plan.md` into the ticket folder. Keep the Delivery Shape section concise (aim for under
40 lines): it states structure, it does not restate the design.

```markdown
# Implementation Plan: [Feature Name]

> Design: design.md
> Research: research-<repo>.md, ...

## Delivery Shape
**First usable milestone**: [Phase N - what a reviewer can see working]
**Phases**: [ordered, one line each: name, goal, depends on]
**PRs**: [PR -> phases it contains, per repo]
**Releases**: [release -> PRs, deployment order, why each stands alone; or "single release"]
**Verification**: [per phase: end-to-end / automated / manual gate]
**Testing strategy**: [what is unit vs integration vs manually verified across phases]

## Phase 1: [Name]

### 1.1 [Change group]
**File**: `path/to/file.ext`
**Action**: [Create | Modify | Delete]
**Details**:
- [Specific change with enough detail to execute]
- [Follow pattern from `path/to/example.ext:L42-L60`]

### Phase 1 Verification
#### Automated:
- [ ] `[test command]` passes
- [ ] `[typecheck command]` passes

#### Manual:
- [ ] [Specific thing to verify]

**Pause for manual verification before proceeding.**

---

## Phase 2: [Name]
...
```

### Step 4: Solution-design-lens antagonistic review

The reviewer is primed with the design + research (the solution-design lens), NOT the plan's own
authoring rationalisation - so it checks the plan against intent, not against itself. It reviews
BOTH the delivery shape (milestone, slicing and phase sizing, PR grouping, release boundaries and
stand-alone safety) and the detailed tasks.

- Read the pertinent CLAUDE.md rules (subagents do not auto-read CLAUDE.md).
- Invoke the `ship-antagonistic-review` skill with ARTIFACT = `plan.md`, LENSES = `design`,
  CONTEXT = the design, the research and any legacy `work-breakdown.md`, and the CLAUDE.md rules
  verbatim.
- Assess findings; incorporate the valid ones into `plan.md`.
- Log to the manifest under `### plan.md`: `- Review [antagonistic:design-lens]: <outcome>`, plus a
  `- Review [manual-cross-thread]: pending` line for any later manual feedback.

### Step 5: Manifest + handoff

- Mark the Create plan stage `done` with the artifact path.
- Legacy manifests only: leave a `done` `Work breakdown` (or `Structure outline`) row untouched;
  set any other status on that row to `n/a` with the artifact column `merged into Create plan`.
- Set `Next` to `/ship-implement-plan` (stage 5 - terminal execution stage).
- Present the plan file location; point the user at the Delivery Shape section as the part worth
  a skim, and remind them the real review happened at design level. Print the handoff block.

## Rules

- **Shape first, then detail.** Decide milestone, slicing, PR grouping and release boundaries before writing file-level tasks. PR grouping and deployment order are separate decisions.
- **Thin slices, session-sized.** Prefer thinner vertical slices over sub-phases; split any phase one `/ship-implement-plan` run cannot finish in a fresh ~150k-token thread.
- **Independent releases stand alone.** A release never depends on a later one. Do not impose release ceremony on work that ships as one PR.
- **This plan is for agent execution.** Write it as instructions, not as a proposal.
- **No open questions.** Every decision was made upstream. If you find an unresolved question, stop and ask before continuing.
- **Reference patterns, don't reinvent.** Point to the design's code snippets. Don't discover new patterns here.
- **Pause between phases.** Every phase ends with verification and an explicit pause for manual checks.
- **No research.** The research is done. If information is missing, the upstream artifacts are incomplete - flag it rather than spawning explorers.
- **Single-threaded stage** - may write `manifest.md`.
- **Stay under 40 instructions total.**
