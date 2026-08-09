# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo status

This repository is currently a **planning-stage scaffold** — there is no source code yet, no build system, and no tests. The only content is `README.md` (placeholder) and `docs/context/`, which holds the design recon and decisions for the "Quartermaster" project. Do not assume application code, package manifests, or CI exist until you've checked — this repo predates all of that.

Because there's nothing to build/lint/test, don't invent commands for them. When implementation starts, this section should be replaced with the real commands.

Note: `.gitignore` is a C++/CMake template (vcpkg, CMakeCache, `.pdb`/`.obj`, etc.) even though no C++ code exists yet — likely added preemptively. Don't treat it as evidence of an existing build system.

## What Quartermaster is

A team token/cost-budget skill family for AI coding agents, targeting **Claude Code and GitHub Copilot CLI only** (not broad multi-tool portability like the `caveman` project it's inspired by). Full context and rationale live in `docs/context/` — read both files before doing any design or implementation work here, since the decisions below are already locked and shouldn't be re-litigated without reason:

- `docs/context/caveman-recon.md` — architecture recon of `JuliusBrussee/caveman`, the project this design borrows patterns from (skill/hook/subagent structure, install mechanism, eval discipline).
- `docs/context/phase2-kickoff.md` — the actual scope, decisions, and build order for Quartermaster.

## Locked decisions (from phase2-kickoff.md)

- **Two separate repos**, not one project:
  - `quartermaster-ledger` — consumption/cost reporting (component 1). Team-wide, read-only, no behavior change. Build first, on Copilot CLI (real token/cost data available via `~/.copilot/session-state/*/events.jsonl`), then port to Claude Code as an *estimate* (no hook-accessible token data there — surface that asymmetry in the UI, don't present both as equally exact).
  - `quartermaster-crew` — subagent-routing agents (component 2), adapted from caveman's `cavecrew` pattern (cheap-model investigator/reviewer subagents with strict typed output contracts, full-model main thread for building). Personal/opt-in, not for team rollout yet.
- Shared "detect Claude Code vs. Copilot CLI" plumbing is **duplicated** across both repos on purpose — no shared-core third repo unless duplication actually becomes painful.
- Build order: **measure → route → compress**. Prose compression (caveman-speak) is deprioritized/optional — see phase2-kickoff.md's "why caveman's content doesn't transfer" for why it's a poor fit for dense engine-architecture conversation.
- A parked, unstarted "skill 2" (team code-style/craftsmanship enforcement) has candidate names **Foreman** or **Smith** but no repo yet — don't create one preemptively.
- This repo itself (`quartermaster`) appears to be the umbrella/planning space — actual implementation is expected to happen in the two child repos above, likely as siblings once created.
