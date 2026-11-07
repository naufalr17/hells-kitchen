# Hells Kitchen 🔪

**Yes, chef.**

Hells Kitchen runs your parallel subagents like a brutal head chef runs a kitchen: the **cheapest capable model per slice**, **git-worktree isolation** so stations never contaminate each other, **evidence or it didn't happen**, and dishes **bounced back until they meet the standard**. No sugar-coated reports — from the chefs to the head chef, or from the head chef to you.

Any agent. Any model. Any provider.

## The problem it fixes

Everyone running parallel agents makes the same two mistakes:

1. **Every subagent runs the flagship model at max effort** — boilerplate, renames, and test scaffolding burn frontier-priced tokens. Your quota dies fast and buys nothing.
2. **Parallel agents share one checkout** — index collisions, half-written files read mid-edit, corrupted work. Then an agent reports "all tests pass" for tests it never ran, and you merge it.

## The system

| Role | Who | Job |
|---|---|---|
| **Manager** | You | Set the order, approve the plan. Never micromanage stations. |
| **Head chef** | Your main agent session | Decompose, write briefs, assign the cheapest model+effort per slice, audit every diff, merge, keep the kitchen board. |
| **Chefs** | Spawned workers | Execute the brief in an isolated worktree, self-QC (bounded — 3 attempts, then an honest blocker report), hand up evidence, die. |

What's inside the SKILL.md:

- **Dispatch decision rule** — what the head chef cooks personally, what gets delegated, and when spawning loses money
- **Pre-dispatch gate** — write-disjoint slices, frozen contracts, mechanical acceptance criteria. Decomposition is where parallelism actually fails
- **Worktree lifecycle** — structural isolation ("conventions are not concurrency control"), cold-kitchen setup, cleanup
- **Brief template** — the chef knows *nothing* except the brief; this is the template that keeps rework near zero
- **Review, audit & merge** — claims vs evidence, a blast-radius proof ladder for risky diffs, a security pass, an escalation ladder for bounced dishes
- **Kitchen board** — a committed `AGENTS.md` every future chef auto-inherits lessons from
- **Arena & swarm recipes** — best-of-N with a private rubric and a fresh-eyes judge; fan-out with write-disjoint slices
- **Adapters** — concrete spawn commands for Hermes, Claude Code, Codex CLI, OpenCode, and a generic fallback

## Install

**skills CLI (Claude Code, Codex, Cursor, OpenCode, and others):**

```bash
npx skills add naufalr17/hells-kitchen
```

**Hermes:**

```bash
hermes skills install naufalr17/hells-kitchen
```

**Manual (any agent that reads SKILL.md):** copy `skills/hells-kitchen/` into your agent's skills directory.

## Philosophy

- **Parallelism buys speed only; model tiering is what stretches your quota.**
- **A chef reporting "blocked, here's why" is a success condition** — cheaper than a confident lie.
- **Review by a smart agent is still fallible; a script that fails loud is not.** Build the verifier before fanning out.
- **Flattery wastes the manager's money.**

## License

MIT
