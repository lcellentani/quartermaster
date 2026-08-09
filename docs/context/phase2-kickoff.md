# Phase 2 kickoff — team token-budget skill

Carried over from recon + planning. Paste this into your first Claude Code / Copilot CLI session on your desktop so nothing gets lost in the handoff.

## Scope decisions made so far
- Targets: **Claude Code** and **GitHub Copilot CLI** only — not caveman's full 30+-tool portability.
- Two skills planned overall, budget first: (1) token/cost budget control, (2) team code-style/craftsmanship enforcement for LLM-generated code (motivated by non-engineers on the team producing code that doesn't match team quality bar) — **skill 2 is still parked, not started.**
- The budget skill splits into **two separate repos**, not one project (see below).
- Build order within the budget skill, in priority: **measure → route → compress**. Compression (caveman-speak) is explicitly last/optional (see "why caveman's content doesn't transfer" below for the full reasoning).

## Repo split & naming (locked)

Two repos, not one:

| Repo | Component | Audience / rollout |
|---|---|---|
| **`quartermaster-ledger`** | Consumption reporting (component 1) | Team-wide. Read-only observability, no behavior change — a good candidate for Copilot CLI's enterprise-managed plugin push once proven. |
| **`quartermaster-crew`** | Subagent-routing agents (component 2) | Personal, opinionated technique. Changes how you work, so it stays an opt-in power tool, not something to blanket-push to the team before it's proven. |

Why split rather than one repo: the two components have genuinely different maturity curves and rollout paths — bundling them means holding back the low-risk one until the experimental one is also team-ready, or shipping experimental behavior to people who didn't ask for it.

The cost of splitting: both repos need the same "detect Claude Code vs. Copilot CLI, write to the right config path" plumbing. Decision: **duplicate it for now**, don't extract a third shared-core repo — that's premature structure for two repos with one maintainer. Revisit extraction only if the duplication starts actually hurting.

Skill 2 (style/craftsmanship, still parked) doesn't have a locked name yet. Candidate personas discussed, to keep the family coherent under the Quartermaster umbrella when it starts: **Foreman** or **Smith**.

## Component 1 — `quartermaster-ledger` (build first, on Copilot CLI)

Copilot CLI already exposes real, server-reported usage data:
- `~/.copilot/session-state/<session-id>/events.jsonl` — `session.shutdown.modelMetrics.*` includes actual per-model token counts and AI-credit cost.
- `/usage` (current session) and `/context` (context-window usage) slash commands.
- Personal hooks live in `~/.copilot/hooks`; `sessionStart`/`sessionEnd` hooks can read and aggregate the events file.

Reference implementations worth reading before building (found via search, unverified beyond their own descriptions — read the code, don't just trust the README):
- `github.com/J-Bax/copilot-token-tracker` — sessionStart/sessionEnd hooks, SQLite aggregation, cross-session dashboard.
- `github.com/devartifex/copilot-cost` — reads Copilot CLI's OpenTelemetry traces instead of the JSONL directly; local-only dashboard.
- `github.com/DamianEdwards/copilot-cli-cost` — real-time statusline cost display.

Target for a first version: a `sessionEnd` hook that appends `{session_id, tokens, cost, timestamp}` to a local log, plus a statusline or `/team-usage` command to surface it. Design the hook to be enterprise-plugin-shippable from day one, since that's the intended rollout path.

**Then port to Claude Code**, accepting it'll be an estimate, not exact — there's no hook-accessible token data there (open issue `anthropics/claude-code#52089`: hooks get session context but no token counts, and nothing is written to an accessible file either). Approach: local tokenizer estimate over what the `Stop` hook can see, same caveat caveman's own `evals/measure.py` carries ("ratios meaningful, absolute numbers approximate"). Surface in the UI that this number is approximate on Claude Code and exact on Copilot CLI — don't let the two look equally authoritative.

## Component 2 — `quartermaster-crew` (build second, on Claude Code)

Direct adaptation of caveman's `cavecrew` pattern (already read in full during recon):
- Investigation/location work → cheap model (haiku), strict output contract (`path:line — symbol — note`), refuses to suggest fixes.
- Review work → cheap model, one-line severity-tagged findings.
- Actual building/architecture work → full model, main thread.

Port to Copilot CLI once proven: it has built-in subagents (`explore`, `task`) and custom agent profiles — contract format not yet reviewed, needs its own read-through when you get there.

### What it actually fixes (and what it doesn't) — from the day-to-day workflow diagnosis

Ludo's current single-agent habit spans Claude Code, Copilot CLI, and Claude.ai chat. Long, hard-to-scan replies come from two different kinds of asks, and they need different fixes — don't build `quartermaster-crew` as a general "shorten replies" tool, or it'll be scoped wrong:

- **Exploration/investigation asks** ("find X, explain how it connects to Y") — genuinely fixed by delegation. A typed-contract investigator subagent keeps the narrated grep/read cycle off the main thread entirely, so it doesn't cost context and doesn't produce a wall of prose in the main session. **This is `quartermaster-crew`'s actual job.**
- **Design/explain/discussion asks** (architecture tradeoffs, decisions) — NOT a delegation problem. The reasoning is inherently main-thread work; there's no subagent to hand a design conversation to. The fix here is a prompting habit — ask for structure ("tradeoffs as a table, decision at top") instead of open-ended ("what do you think") — not infrastructure. Try this separately from building `quartermaster-crew`, so the two fixes don't get credited to each other and you can actually tell which is doing the work.

## Why caveman's compression rules don't transfer to this context

Kept here because it's the reasoning behind deprioritizing component 3 (prose compression) — worth having on hand if the question comes up again, e.g. when pitching this to the team:

- Compression rules are calibrated for chatty, generalist dev conversation. Dense engine-architecture discussion is already terse — there's less filler to cut, and more risk of the skill going net-negative once its own ~1-1.5k token overhead is counted (their own "honest numbers" doc admits this happens on already-terse workloads).
- Caveman's own "Auto-Clarity" escape hatch (drops compression for security warnings, irreversible actions, and ambiguity-creating multi-step sequences) would trigger constantly on Frostbite-adjacent work — memory safety, threading hazards, platform certification, backward-compat constraints. If most real conversations would trip that hatch anyway, the feature barely engages where it matters.
- Caveman only ever compresses output *prose* — code, commands, and errors are explicitly exempted, never touched. For a large C++ engine codebase the real token cost is context volume (files read, tool-result size) and generated code/diffs, not chat filler — caveman optimizes an axis that was never going to bust the budget here.

## Not started yet

- Component 3 (prose compression, under whichever repo it ends up in) — deprioritized, build only if components 1 and 2 don't move the number enough.
- Skill 2 (style/craftsmanship enforcement) — parked until the budget skill is prototyped. When it starts: the hard part isn't the hook/subagent plumbing (borrow `cavecrew-reviewer`'s contract shape), it's turning the team's implicit conventions into concrete, example-anchored do/don't rules the way caveman did per intensity level — that's the work that actually determines whether it's useful.

## Reference

Full architecture recon of `JuliusBrussee/caveman` (skill/hook/subagent patterns, install mechanism, eval discipline) was written up separately — pull it back up if a design question needs grounding in how they solved it.
