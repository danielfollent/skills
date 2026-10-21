# Claude Code Skills

A collection of [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) that make up the **Ship pipeline** for structured software delivery.

## Quick Reference

| Skill | Description | Install |
|-------|-------------|---------|
| `/ship` | Pipeline dispatcher — runs the next stage from the manifest | `npx skills@latest add dfollent/skills/ship` |
| `/ship-generate-questions` | Repo scoping + intent-neutral research questions (stage 1) | `npx skills@latest add dfollent/skills/ship-generate-questions` |
| `/ship-research-codebase` | Objective codebase research from questions (stage 2) | `npx skills@latest add dfollent/skills/ship-research-codebase` |
| `/ship-solution-design` | Solution design with decision-tree question walk + antagonistic review (stage 3) | `npx skills@latest add dfollent/skills/ship-solution-design` |
| `/ship-create-plan` | Delivery shape + detailed tactical plan, hardened by antagonistic review (stage 4) | `npx skills@latest add dfollent/skills/ship-create-plan` |
| `/ship-implement-plan` | Phase-by-phase plan execution with verification + review of the diff (stage 5) | `npx skills@latest add dfollent/skills/ship-implement-plan` |
| `/ship-antagonistic-review` | Adversarial per-lens review of a design, plan or diff (used by stages 3–5) | `npx skills@latest add dfollent/skills/ship-antagonistic-review` |

Install all skills at once:

```bash
npx skills@latest add dfollent/skills
```

## Requirements

Only Claude Code. The Ship skills use built-in tools and `general-purpose` subagents; no custom subagents, MCP servers or other skills are needed. Install the whole set so the skills sit side by side in the same skills directory: the stages read the shared manifest conventions from `ship/manifest-format.md` and call `ship-antagonistic-review` for adversarial review.

A ticket tracker integration (e.g. a Jira MCP server) is optional. Without one, paste the ticket text when a stage asks for it.

## The Ship pipeline

Skills that break delivery into focused stages. Each stage runs in its own context window, produces a static markdown artefact in a per-ticket `<TICKET>-ship/` folder, and feeds into the next. Human review concentrates where it has the most leverage.

The `/ship` dispatcher is an on-rails orchestrator: it resolves the active per-ticket manifest and runs **exactly one** stage per invocation. Every `ship-` skill is also independently runnable on its own.

### The flow

**1. `/ship-generate-questions`** — Scopes which repos in a multi-repo workspace the ticket touches (it legitimately sees the ticket), then writes one intent-neutral research-questions file per repo plus the per-ticket manifest.

**2. `/ship-research-codebase`** — Takes only the questions (never sees the ticket). Researches each repo in-thread, or fans out parallel agents (one per repo by default). Outputs an objective `research-<repo>.md` per repo. No opinions.

**3. `/ship-solution-design`** — Takes research + ticket. Produces a solution design covering the coverage map, driven by an interactive decision-tree question walk and hardened by tiered antagonistic review. First review point.

**4. `/ship-create-plan`** — Takes design + research. Decides the delivery shape first (first usable milestone, session-sized vertical slices, PR grouping, release boundaries, phase order and verification), then expands it into a detailed tactical plan the agent executes, hardened by a design-lens antagonistic review. Skim the Delivery Shape section; no full review needed.

**5. `/ship-implement-plan`** — Execute the plan, phase by phase, with restraint (reuse before writing, no unrequested abstractions, tests that pin behaviour), verification and a pause for manual checks between phases. Ends with a comment audit and an antagonistic review of the diff before the final commit.

**`/ship-antagonistic-review`** — The shared reviewer. Spawns fresh `general-purpose` subagents that try to break an artifact through one lens each (product, architecture, security, performance, reliability, data, or design-conformance) and return structured findings. Called by stages 3, 4 and 5; also runnable on any design, plan or diff.

### Why it works

- **You review the design and skim the plan's delivery shape, not 1000 lines of plan or code.** That's where leverage is highest.
- **Each skill stays small (40-55 instructions).** Smaller prompts = fewer skipped steps.
- **Research is unbiased.** The questions stage sees the intent; the research stage doesn't. Prevents cherry-picking facts that support what the model already wants to build.
- **Every stage produces a standalone markdown doc** in the per-ticket folder. Resume from any point in a fresh session.
- **Antagonistic review** stress-tests the design and plan through adversarial single-lens passes before any code is written, and the diff before it is committed.

### Skipping steps

Not every task needs all five stages. Use your judgement:

- **Small bug fix?** Skip straight to implementation.
- **Well-understood change?** Start at `/ship-solution-design` or `/ship-create-plan`.
- **Complex feature in unfamiliar code?** Run the full flow via `/ship`.

## Acknowledgements

The coding workflow is informed by [Dex Horthy's](https://github.com/humanlayer) CRISPY methodology and the evolution from Research-Plan-Implement (RPI). The solution-design question walk is inspired by [Matt Pocock's grill-me](https://github.com/mattpocock/skills/blob/main/grill-me/SKILL.md) (MIT). Several skills are derived from or inspired by [HumanLayer's](https://github.com/humanlayer/humanlayer) Claude Code commands (Apache 2.0). See [NOTICE](NOTICE) for full attribution.
