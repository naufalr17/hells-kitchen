---
name: hells-kitchen
description: "Use when parallelizing work across subagents. Yes, chef: the cheapest capable model per slice, git-worktree isolation, evidence or it didn't happen, no sugar-coated reports. Any agent, any model, any provider."
metadata:
  tags: [orchestration, subagents, worktree, cost-optimization, parallelism, verification]
---

# Hells Kitchen

Run parallel agents like a brutal head chef runs a *brigade de cuisine*. Roles:

- **Manager** (the human): sets the order, approves the plan, never micromanages stations.
- **Head chef** (the main session): owns the kitchen — decomposes, writes briefs, assigns the cheapest model+effort that can nail each slice, audits, merges, maintains the kitchen board.
- **Chefs** (spawned workers): execute the brief, self-QC, hand up evidence, then die. A chef knows NOTHING except its brief.

Optimal = right-sized spend per slice, never "every agent on the flagship model at max effort."

## Dispatch decision rule (apply in order)

1. **Judgment-critical** (architecture, security-sensitive code, gnarly bugs, hotfixes, deploys, final review/merge) → head chef cooks personally. Never delegate.
2. **In-session subagents that inherit the parent model** (Claude Code Task tool, Hermes delegate_task, etc.) burn parent-priced tokens and bloat parent context. Use only for short read-only subtasks where spawn overhead exceeds the model-rate saving.
3. **Meaty slice a lower tier can handle** → spawn a separate agent process at a cheaper model/effort, working directory set INSIDE its own git worktree, in the background. Each spawn is a fresh conversation — mixed models/efforts carry zero prompt-cache penalty. (Commands per agent: see Adapters.)
4. **Small tasks: don't spawn.** Spawn overhead is thousands of tokens before any work. The pattern wins only on meaty slices (a feature, a page, a module) and loses when writing the brief takes longer than doing the work.

## Model tiers (one billing pool per provider — tiering stretches it, parallelism only buys speed)

| Tier | Class (any provider) | Assign |
|---|---|---|
| Cheap chef | smallest capable model, minimal reasoning | Renames, boilerplate, test scaffolding, doc updates, grep-and-summarize research |
| Mid chef | mid model, low/medium reasoning | Standard features, CRUD endpoints, UI components from a clear spec |
| Flagship chef | frontier model, high reasoning | Architecture, hard debugging, security, review/merge (usually the head chef itself) |

Model AND effort are independent dials — a mid model at high reasoning often beats a frontier model at minimal reasoning on algorithmic work, at lower cost. Downshift both only when the slice is mechanical.

## Pre-dispatch gate (decomposition is where this pattern actually fails)

Do not spawn anything until every check passes:

- [ ] **Slices are write-disjoint** — no two chefs own the same file. Shared write target = merge conflict by design; re-cut or serialize. "Separate before serializing": eliminate sharing first; a lock or sequential phase only when one shared writer is a real invariant.
- [ ] **Contracts frozen** — every interface between slices (API shape, types, file formats) is written in the briefs. Chefs build to the contract, never negotiate it.
- [ ] **Acceptance criteria are mechanical** — commands with exit codes, rendered screenshots, observable outputs. Never adjectives ("clean", "nice"). A vague criterion invites the chef to grade its own essay.
- [ ] **Tier justified per slice** — mechanical→cheap, spec'd feature→mid, judgment→head chef. Written down, one line each.
- [ ] **Each slice is a verifiable unit** — ends in a state a command can check. Don't advance a dependent wave until the current unit is green; a break caught at its unit is cheap, a break found after a batch is buried under later work.

## Worktree lifecycle (make isolation structural, not instructional)

Conventions are not concurrency control. A brief saying "stay in your worktree" is a wish; structure is enforcement:

1. Per code-editing chef: `git worktree add ../wt-<slice> -b wt/<slice>` (read-only chefs need none).
2. **Spawn the chef with its working directory inside the worktree** — its whole world starts there. (Some agents enforce isolation natively; see Adapters.)
3. A fresh worktree is a cold kitchen: no installed deps, no gitignored env files. The brief MUST include setup (install command, which env files to copy from the main checkout) or the chef cannot verify anything — and an unverifiable chef tends to report done anyway.
4. Done: review diff → audit → merge into the feature branch (NEVER push/merge to master/main) → `git worktree remove ../wt-<slice>` → `git worktree prune`.

## Brief template (the brief is the chef's entire world)

- Goal + mechanical acceptance criteria (exact commands + expected outcomes)
- Worktree path + setup steps (deps, env files, how to start the app, how to tell it's ready)
- Exact files in scope; frozen contracts for everything at the slice boundary (mock at the boundary, don't build the other side)
- Conventions: a reference file to imitate beats prose rules
- **Fix-loop, bounded:** verify against the criteria; on failure fix and re-verify, max 3 attempts; still red → STOP and report the blocker honestly with what was tried. A chef reporting "blocked, here's why" is a success condition — cheaper than a confident lie.
- Guardrails: think before coding; simplest implementation that passes — no speculative abstractions or unrequested features; never report untested work as complete; evidence over adjectives ("ran X, saw Y"); state what was NOT verified; **no sugar-coating** — report failures, uncertainty, and cut corners bluntly, never round a partial result up to "done"
- Required output: commit on the branch + short summary of what was done, what was verified and how, **and anything surprising found** — the summary is the only artifact that outlives the chef

Tight briefs are the token lever: rework from a vague brief costs more than any model downshift saves. Ambiguity doesn't make a chef stop — it makes it decide, confidently and sometimes wrongly.

## Review, audit & merge (head chef — claims are not evidence)

1. Read the chef's summary as a **claim**. Re-run the acceptance checks yourself; for UI, look at screenshots at mobile and desktop widths.
2. **Risky diffs get the blast-radius treatment:** find the ONE fact the change is safe because of, then push it down the proof ladder as far as is cheap — (a) asserted, (b) pointed at file:line, (c) walked the failure path, (d) ran a script against the real code, (e) reproduced in the running app — and state where it stopped. A writeup that merely sounds right is worthless; prove the one fact by running code.
3. **Security — two cadences.** Per-merge quick scan, never skipped (near-free): no secrets/credentials in the diff, no obviously unvalidated input at a trust boundary. The DEEP audit runs at **checkpoints**, never per feature — auditing every small diff burns tokens on noise while missing cross-cutting flaws that only exist once pieces integrate. At plan time the head chef proposes the audit checkpoints and the manager confirms them — ask, don't assume. Default checkpoints to propose: end of each wave/phase; before any deploy or release; after any wave that touched auth, payments, uploads, AI endpoints, or another trust-boundary surface. Run checkpoint audits with a dedicated security-audit skill when one is installed (e.g. Cloudflare's security-audit skill: scoped recon → adversarial review → validation → structured report). Write confirmed findings to the kitchen board so future chefs don't reintroduce them.
4. Merge in the order that proves the work (failing test → fix; scaffold → feature). Each merge lands green before the next.
5. **Escalation ladder:** 1st failure → one corrective re-brief to the SAME tier (rework on cheap tokens is the point). 2nd failure of the same slice → the slice needed more judgment than the tier has: escalate one tier up or cook it personally. Never re-dispatch a third time at the same tier.
6. **Serve with no sugar-coating.** The report to the manager states plainly: what works (with evidence), what is broken or unverified, what was cut and why, what it cost. Never inflate progress, never bury a failure in positives — a blunt bad report beats a pretty false one. Disagree with the manager's ideas when they're wrong, with reasons; flattery wastes the manager's money.

## Kitchen board (shared notes all chefs read)

Keep a committed `AGENTS.md` at the repo root — most agent CLIs auto-load it (or an equivalent like `CLAUDE.md`), and every worktree contains it. That file IS the board: conventions, frozen contracts, security standards, lessons from previous waves ("X pattern has a quirk — don't copy it"). When a chef's summary surfaces something surprising, write it to the board — every FUTURE chef inherits it free, with zero brief bloat. Updates propagate to newly spawned chefs only; update the board between waves, not during.

## Build the verifier before fanning out

If the repo has no scripted way to prove the behavior being built (test harness, drive-the-app script, curl-able endpoint + seed data), build that lever FIRST — head chef or a mid chef, once — and reference it in every brief. Executable verification turns "head chef's taste" into instruments; review by a smart agent is still fallible, a script that fails loud is not. Launch steps must reach a usable state from empty (migrations, seed data); "a process answers" is not "ready".

## Recipes

- **Arena (best-of-N), for judgment-sensitive work only:** (1) Frame: one shared prompt (the prompt is the contract) + a private rubric of 3-6 gradeable criteria — candidates see only the task, never the rubric. (2) Fan out N chefs, own worktree/output dir each, each returning the artifact + a rationale naming alternatives considered and rejected. A candidate that produces nothing → proceed with N-1, note the dropout. (3) Judge with fresh eyes against the rubric — a separate spawned session that authored no candidate; a judge from a different model family is even better when you have multiple providers (authorship bias is the enemy; fresh context kills most of it). (4) Pick the strongest base, graft the best parts of the losers, verify the synthesis against the rubric. Same model N times when work is generation-bound; different models/efforts when it's judgment-sensitive.
- **Swarm (fan-out):** N chefs on DIFFERENT write-disjoint slices, one aggregated report; code-editing workers get worktrees.

## Guard the head chef's context

Route bulk into chefs; keep summaries in the parent thread, never raw payloads. Don't read chef transcripts — read their diffs and summaries. Cap files per wave. A bloated head-chef context degrades the judgment everything else depends on.

## Adapters (map the primitives to your agent)

The pattern needs four primitives: **spawn headless** with a prompt, **pick model**, **pick reasoning effort**, **set working directory**. Flags drift between versions — verify against `--help` before relying on them.

- **Hermes:** `hermes chat -q "<brief>" -m <model> --reasoning <none|minimal|low|medium|high>`, run with the worktree as working directory, in the background.
- **Claude Code:** `claude -p "<brief>" --model <model> --worktree <name>` — worktree creation AND isolation enforcement are built in (file edits outside the worktree are blocked). In-session Task-tool subagents inherit the parent's billing context; prefer separate processes for tiering.
- **Codex CLI:** `codex exec "<brief>" -m <model> -c model_reasoning_effort="<low|medium|high>"`, launched from the worktree directory.
- **OpenCode:** `opencode run "<brief>" --model <provider/model>`, launched from the worktree directory.
- **Anything else:** any agent with a non-interactive mode + model flag can be a chef; the head chef only needs its exit, its diff, and its summary.

## Pitfalls

- Chef summaries are self-reports. A chef claiming "all tests pass" may have run nothing — the merge gate re-verifies, always.
- Fake parallelism: slices that touch the same files serialize anyway; you pay spawn overhead for zero concurrency. The pre-dispatch gate exists for this.
- An unbounded fix-loop at a cheap tier is a token incinerator that pressures the chef into rounding failure up to "done". Bound it (3 attempts) and reward honest blocker reports.
- Changing which skills/plugins are installed usually edits the agent's system prompt and invalidates its prompt cache once — batch such changes, never spread them over a session.
