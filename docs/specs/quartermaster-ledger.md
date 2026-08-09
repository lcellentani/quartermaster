# Quartermaster Ledger (component 1) — SPEC

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Date** | 2026-08-09 |
| **Plan** | Not yet planned |
| **Supersedes** | N/A |

---

## 1. Problem

Team members using Claude Code and Copilot CLI have no visibility into per-session token/cost consumption. Copilot CLI already reports real usage data server-side (`~/.copilot/session-state/<session-id>/events.jsonl`), but nothing captures or surfaces it; Claude Code exposes even less. There's no local record to look back on and no shared way to see what agent usage is actually costing.

## 2. Goals

- A Copilot CLI `sessionEnd` hook appends one JSONL record per session (`{session_id, tokens, cost, timestamp}`) to a local log, sourced from `events.jsonl`'s `session.shutdown.modelMetrics.*`.
- A statusline surfaces running token/cost usage during an active Copilot CLI session.
- The same hook + log + statusline pattern is ported to Claude Code, using a local tokenizer estimate over whatever the `Stop` hook can see (Claude Code exposes no hook-accessible token counts — tracked upstream as `anthropics/claude-code#52089`).
- The UI visibly distinguishes Copilot CLI's exact numbers from Claude Code's estimated numbers — the two are never presented as equally authoritative.
- The hook/plugin is structured from day one to be shippable as a Copilot CLI enterprise-managed plugin, since that's the intended team rollout path.

## 3. Non-Goals

- Cross-machine/team log aggregation — genuinely unresolved, see Open Questions; not designed or built in this spec.
- Component 2 (`crew/`, subagent routing) — separate component, own spec.
- Skill 2 (style/craftsmanship enforcement, candidate names Foreman/Smith) — parked, not started.
- Prose compression (caveman-speak) — deprioritized per the kickoff doc's build order (measure → route → compress); build only if components 1 and 2 don't move the number enough.
- SQLite or any queryable storage backend — v1 uses a flat JSONL append log only.
- `/team-usage` (or any on-demand) slash command — v1 ships a statusline only; an explicit query command is additive future scope.

## 4. Design

### Key Decisions

- **Build on Copilot CLI first, port to Claude Code second** — Copilot CLI has real server-reported usage data to build against; Claude Code has no hook-accessible token data, so starting there would mean building an estimate with no ground truth to calibrate it against.
- **Flat JSONL append log, not SQLite** — v1 has one maintainer and no query workload beyond simple sums; JSONL also matches the format of the `events.jsonl` source it reads from. Revisit only if aggregation math becomes genuinely painful to do over JSONL.
- **Statusline as the v1 surface, not `/team-usage`** — the kickoff doc's own "target for a first version" language names a statusline first; an on-demand command is additive, deferred until an always-visible surface proves insufficient alone.
- **Claude Code numbers ship explicitly as estimates, with mandatory UI labeling** — alternative (treating Claude Code's tokenizer-derived estimate as equal-precision to Copilot CLI's server-reported numbers) was rejected: it would let the team draw false confidence from a number that isn't actually comparable. Mirrors caveman's own `evals/measure.py` caveat: "ratios meaningful, absolute numbers approximate."
- **Hook designed for enterprise-managed plugin packaging from day one** — the intended team rollout is a Copilot CLI plugin push; retrofitting packaging constraints onto the hook's install/config surface later would mean redesigning it after the fact.

### How It Fits Existing Systems

- Copilot CLI personal hooks (`~/.copilot/hooks`), specifically `sessionStart`/`sessionEnd`, reading `~/.copilot/session-state/<session-id>/events.jsonl`.
- Claude Code's `Stop` hook — the only session-end lifecycle point available, but without token-count data.
- The "detect Claude Code vs. Copilot CLI, write to the right config path" plumbing described in `phase2-kickoff.md`'s 2026-08-09 revision, which combined `ledger/` and `crew/` into one repo specifically so this plumbing lives once instead of being duplicated. **Note:** `CLAUDE.md`'s repo-structure section still describes the older two-repo split — this spec follows the newer, more specific `phase2-kickoff.md` decision; reconciling `CLAUDE.md` is outside this spec's scope but should happen separately.
- Reference implementations named in the kickoff doc (`J-Bax/copilot-token-tracker`, `devartifex/copilot-cost`, `DamianEdwards/copilot-cli-cost`) — worth a code read before implementation starts; not yet done as part of this spec.

## 5. Open Questions

- [ ] How does per-user, per-machine hook logging become genuinely "team-wide" observability? No aggregation/rollup mechanism is defined — explicitly left open rather than assumed.
- [ ] Which local tokenizer/estimation method should the Claude Code `Stop`-hook estimate use, and how should its error margin be communicated in the UI?
- [ ] Where does the local JSONL log live (path, one file per session vs. one running file), and what's the retention/rotation policy?
- [ ] What does "enterprise-managed plugin push" concretely require of the hook's packaging (manifest format, permissions, config surface)? Needs a read of Copilot CLI's plugin docs.
- [ ] Should `CLAUDE.md`'s repo-structure section be updated to match `phase2-kickoff.md`'s combined-repo revision, to avoid the two documents disagreeing?

---

## Status Legend

| Status | Meaning |
|--------|---------|
| Draft | Still being shaped; open questions may exist |
| Ready | Open questions resolved; safe to run `/plan-draft` |
| Implemented | All linked plan phases are Done |
| Superseded | Replaced by a newer spec — link it above |
