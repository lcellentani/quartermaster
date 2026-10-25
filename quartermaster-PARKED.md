# PARKED: Quartermaster

| Field | Value |
|-------|-------|
| **Status** | Parked at planning stage |
| **Date** | 2026-10-25 |
| **Active period** | 2026-08-09 to 2026-08-10 (7 commits) |

## What it was

A planned token/cost-budget skill family for Claude Code and GitHub Copilot CLI. Build order: **measure, then route, then compress**.
- **Ledger:** personal per-session token/cost log with a statusline. Exact numbers on Copilot CLI, estimated on Claude Code, and the UI must never present them as equally exact.
- **Crew:** cheap-model investigator/reviewer subagents with typed output contracts.
- A third idea, a team code-quality enforcement skill, was parked before it started.

## What got built

No code. The repo holds the recon of the `caveman` project, the Phase 2 kickoff decisions, the Ledger SPEC (Draft, not planned) and the slash-command workflow.

## Why I parked it

My company introduced a well structured tool to monitor AI consumption on multiple dimensions so there was no real reason to continue this project. I learned interesting concepts about what tools like Claude Code and Github Copilot offer to track usages and I completed a study of tools I selected as inspiration.

## What I learned

- Claude Code's `Stop` hook gives a transcript path with real per-turn usage data, but output tokens were reported as undercounted (upstream issue `anthropics/claude-code#27361` at the time; recheck).
- Starting with one repo and splitting later is cheaper than the reverse.

## What would make me resume

Folding the Ledger into `kitbash` as a plugin rather than a separate repo, if I want cost visibility in my own workflow. Start from `docs/specs/quartermaster-ledger.md`.
