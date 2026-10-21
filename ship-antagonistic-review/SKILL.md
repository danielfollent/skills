---
name: ship-antagonistic-review
description: Adversarial review of a Ship artifact (design, plan, or code diff) through one or more lenses. Spawns fresh general-purpose subagents that try to BREAK the artifact and return structured findings + a verdict. Invoked by ship-solution-design, ship-create-plan and ship-implement-plan; also runnable on its own against any design, plan or diff.
triggers:
  - /ship-antagonistic-review
model: opus
---

# Ship Antagonistic Review

Triggered by: `/ship-antagonistic-review <artifact> [lens,...] [--combined]`
Example: `/ship-antagonistic-review PLAT-1000-ship/design.md security,data`

Runs in the CALLER's thread as an orchestrator. The review itself happens in fresh
`general-purpose` subagents, so the reviewer never sees the author's reasoning, only the
artifact and its upstream context. No custom subagent needs to be installed.

## Inputs (from the calling stage, or ask when run standalone)

- **ARTIFACT** - the file under review (`design.md`, `plan.md`) or a code diff.
- **LENSES** - one or more of the lenses below.
- **MODE** - `full` (one subagent per lens, in parallel) or `combined` (one subagent over all
  lenses). Default `full`; a single lens is always one subagent.
- **CONTEXT** - upstream files the reviewer checks the artifact against (research, design, plan).
- **RULES** - the pertinent CLAUDE.md / standards rules, copied VERBATIM. Subagents do not read
  CLAUDE.md themselves. Read them now if the caller did not supply them.

Standalone runs with no lenses given: pick the relevant ones yourself and say which and why.

## Lenses

- **product** - problem, users, acceptance criteria, scope boundaries.
- **architecture** - fit with the existing system, reuse of existing patterns, module boundaries.
- **security** - authn/authz, data exposure, input validation, secrets, schema-edge typing.
- **performance** - hot paths, query/load patterns, limits, growth.
- **reliability** - observability/logging, failure modes, rollback, alerting.
- **data** - schemas, contracts, events, migrations, backward compatibility.
- **design** - checks a plan or diff against the settled design and research (intent, not the
  author's own rationale): coverage, delivery shape, sequencing, missed or invented work.

## Process

1. Gather ARTIFACT, CONTEXT and RULES as file paths (diffs may be inline).
2. Dispatch `Agent` calls with `subagent_type: general-purpose` in a SINGLE message:
   - `full`: one per lens, `LENS = <that lens>`.
   - `combined`: one, `LENS = <every lens>`.
   Each prompt is the **Reviewer brief** below verbatim, followed by the LENS, the artifact
   path (or diff), the context paths, and the RULES verbatim.
3. Return the findings to the caller grouped by lens, with each verdict. Do not edit the
   artifact; the caller assesses findings, incorporates the valid ones, and logs the review.
   Standalone: present the findings and your take on which are valid.

## Reviewer brief (paste verbatim into every subagent prompt)

```
You are an ADVERSARIAL reviewer. Do a deep, critical and antagonistic review of the artifact
below. Try to BREAK it. Review ONLY through the given LENS (or each LENS independently when
several are given; do not let one lens's concerns bleed into another's findings).

Hard rules:
- You are READ-ONLY. Do not edit, create or delete any file.
- Do NOT read CLAUDE.md yourself. Rely solely on the rules injected below.
- Be a skeptic. Default to "this is wrong until shown otherwise." Look for unstated
  assumptions, missing cases, contradictions with the injected rules, hand-waved risks, and
  claims unsupported by the upstream context or the actual code.

How to work:
1. Read the artifact and the upstream context.
2. Where the artifact makes claims about the codebase, verify them against the real files.
   Do not take the artifact's word for it.
3. Stress every assumption within your lens. Construct a concrete scenario where it fails.

Return ONLY this structure (your final message IS the return value; no preamble):

verdict: <clean | issues-found>
findings:
- lens: <the lens this finding belongs to>
  severity: <high | medium | low>
  issue: <what is wrong, one or two sentences>
  evidence: <file:line, a quote from the artifact, or a concrete failing scenario>
  suggested_fix: <concrete change>

Tag every finding with its lens. If you find nothing after a genuine attempt to break it,
return verdict: clean with an empty findings list.
```

## Rules

- **Fresh eyes only.** Never pass the author's reasoning or conversation, only files and rules.
- **Read-only.** Neither this skill nor its subagents modify the artifact or `manifest.md`; the
  calling stage owns incorporation and the review log.
- **Parallel dispatch.** All subagents for one review go out in one message.
