# Quartermaster Ledger (component 1) — SPEC

| Field | Value |
|-------|-------|
| **Status** | Draft |
| **Date** | 2026-08-09 |
| **Plan** | Not yet planned |
| **Supersedes** | N/A |

---

## 1. Problem

Team members using Claude Code and Copilot CLI have no visibility into their own per-session token/cost consumption. Copilot CLI already reports real usage data server-side (`~/.copilot/session-state/<session-id>/events.jsonl`), but nothing captures or surfaces it; Claude Code exposes even less. Without a local record to look back on, there's no way for someone to notice their own costly patterns session over session and adjust — nothing to self-assess against.

## 2. Goals

- A Copilot CLI `sessionEnd` hook appends one JSONL record per session (`{session_id, tokens, cost, timestamp}`) to a local log, sourced from `events.jsonl`'s `session.shutdown.modelMetrics.*`.
- A statusline surfaces running token/cost usage during an active Copilot CLI session.
- The same hook + log pattern is ported to Claude Code. The `Stop` hook's `transcript_path` field points to the session's JSONL transcript, which carries real per-turn `message.usage` data (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) from the actual API responses — input/cache figures are treated as exact; output tokens need a correction for a known, open upstream bug (`anthropics/claude-code#27361`) that undercounts them by omitting the final `message_stop` event — corrected by counting `last_assistant_message` per turn via Anthropic's token-counting API rather than trusting the transcript's mid-stream snapshot. Because `Stop` fires once, at session end, Claude Code's statusline shows the **last completed session's** numbers rather than live/in-progress usage — blank until the first session ends, then labeled e.g. `Last session: ~12k tok (est.)` to flag both the staleness and the still-corrected nature of the output-token figure.
- The UI visibly distinguishes Copilot CLI's exact numbers from Claude Code's estimated numbers — the two are never presented as equally authoritative.

## 3. Non-Goals

- Cross-machine/team log aggregation, or any shared/rolled-up view of usage — this is a personal, per-machine tracking tool by design, not a team reporting surface.
- Component 2 (`crew/`, subagent routing) — separate component, own spec.
- Skill 2 (style/craftsmanship enforcement, candidate names Foreman/Smith) — parked, not started.
- Prose compression (caveman-speak) — deprioritized per the kickoff doc's build order (measure → route → compress); build only if components 1 and 2 don't move the number enough.
- SQLite or any queryable storage backend — v1 uses a flat JSONL append log only.
- `/team-usage` (or any on-demand) slash command — v1 ships a statusline only; an explicit query command is additive future scope.

## 4. Design

### Key Decisions

- **Scoped as a personal tracking facility, not team reporting** — the value this delivers is a teammate noticing their own costly patterns session over session and self-correcting; it's not a shared dashboard. Cross-machine/team aggregation is out of scope by design (see Non-Goals), not merely deferred.
- **Build on Copilot CLI first, port to Claude Code second** — Copilot CLI has real server-reported usage data to build against; Claude Code's equivalent (`transcript_path`'s `message.usage` data) has a known accuracy gap on output tokens (see below), so starting there would mean building against a source that's itself partly unreliable, with no clean ground truth to calibrate against.
- **Claude Code token counts are read from `transcript_path`, not computed by a from-scratch tokenizer** — the `Stop` hook's `transcript_path` field points to the session JSONL, which carries real per-turn `message.usage` data from the API. Input and cache token fields are used as-is (exact); output tokens require a workaround for a known open upstream bug (`anthropics/claude-code#27361`) that undercounts them by roughly 2x in reported cases, by omitting the final `message_stop` event. Running a local tokenizer over the whole conversation with no reference to the real API-reported numbers at all was considered and rejected: throwing away the transcript's exact input/cache data to re-estimate everything from scratch would make the number less accurate, not more, for most of the total. (The spec's earlier framing — that Claude Code exposes no hook-accessible token data at all, per `anthropics/claude-code#52089` — was based on a request for a ready-made `usage` field on hook stdin, closed as not planned; the raw data is available via `transcript_path`, just not that conveniently.)
- **Output-token correction calls Anthropic's token-counting API on `last_assistant_message`, rather than a local approximation tokenizer** — Claude doesn't publish an official offline tokenizer, so a local stand-in (e.g. tiktoken) would just add a second layer of estimation on top of the correction. Calling the Messages API's token-counting endpoint per session-end gives a count sourced directly from Anthropic, at the cost of a network call and API credentials being available to the `Stop` hook — see Open Questions for how the hook authenticates and what happens if the call fails or the machine is offline. The resulting number stays under a single blended `(est.)` label rather than being split by field, consistent with the labeling decision above.
- **Flat JSONL append log, not SQLite** — v1 has one maintainer and no query workload beyond simple sums; JSONL also matches the format of the `events.jsonl` source it reads from. Revisit only if aggregation math becomes genuinely painful to do over JSONL.
- **Statusline as the v1 surface, not `/team-usage`** — the kickoff doc's own "target for a first version" language names a statusline first; an on-demand command is additive, deferred until an always-visible surface proves insufficient alone.
- **Claude Code numbers ship explicitly as estimates, with mandatory UI labeling** — alternative (treating Claude Code's numbers as equal-precision to Copilot CLI's server-reported numbers) was rejected: it would let the team draw false confidence from a number that isn't actually comparable, even though most of it (input/cache tokens) is in fact exact — the corrected output-token figure is enough to keep the whole number labeled as an estimate rather than split-labeling individual fields. Mirrors caveman's own `evals/measure.py` caveat: "ratios meaningful, absolute numbers approximate."
- **Claude Code's statusline reflects the last completed session, not live usage** — `Stop` is the only session-lifecycle hook Claude Code exposes, and it fires once, at the end. Accumulating a true live estimate would require additional per-turn hooks (`UserPromptSubmit`, `PreToolUse`/`PostToolUse`), which was rejected for v1 as unnecessary complexity beyond what the correction approach above already requires. The statusline stays hidden until the first session ends, then shows a "last session" snapshot with explicit staleness + estimate labeling.
- **Hook packaged as a self-service Copilot CLI marketplace plugin from day one** — distribution follows the same `marketplace.json`-backed marketplace pattern already used for this org's other Copilot CLI skills/agents: users add the marketplace once (`copilot plugin marketplace add <org>/<repo>`), then install the ledger plugin from it like any other. This is distinct from GitHub's separate "enterprise-managed plugins" feature (admin-pushed via `managed-settings.json` in the enterprise's `.github-private` repo, requires Copilot Business/Enterprise licensing) — that mechanism is not in scope here. Retrofitting the marketplace packaging shape onto the hook's install/config surface later would mean redesigning it after the fact.

### How It Fits Existing Systems

- Copilot CLI personal hooks (`~/.copilot/hooks`), specifically `sessionStart`/`sessionEnd`, reading `~/.copilot/session-state/<session-id>/events.jsonl`.
- Claude Code's `Stop` hook — the only session-end lifecycle point available. It passes `transcript_path` (the session's JSONL transcript, carrying real per-turn `message.usage` data) and `last_assistant_message` (the final turn's text, documented as more reliable than the transcript file for that turn since the transcript write lags). Known limitation: `anthropics/claude-code#27361` (open) — the transcript never logs the final `message_stop` event, so `output_tokens` is a mid-stream snapshot, undercounting real usage by roughly 2x in reported cases.
- Claude Code's `statusLine` field in `settings.json` — the config surface that renders the statusline; it can re-run on its own refresh cadence, but between `Stop` events it's reading the same static "last session" log entry, so cadence doesn't change the design. Copilot CLI's equivalent statusline mechanism (plugin API, hook type, or config file) is unresearched — see Open Questions.
- The "detect Claude Code vs. Copilot CLI, write to the right config path" plumbing described in `phase2-kickoff.md`'s 2026-08-09 revision, which combined `ledger/` and `crew/` into one repo specifically so this plumbing lives once instead of being duplicated. **Note:** `CLAUDE.md`'s repo-structure section still describes the older two-repo split — this spec follows the newer, more specific `phase2-kickoff.md` decision; reconciling `CLAUDE.md` is outside this spec's scope but should happen separately.
- Reference implementations named in the kickoff doc (`J-Bax/copilot-token-tracker`, `devartifex/copilot-cost`, `DamianEdwards/copilot-cli-cost`) — worth a code read before implementation starts; not yet done as part of this spec.
- Copilot CLI's plugin marketplace mechanism — a `marketplace.json` in `.github/plugin/` (or `.claude-plugin/`) of the distribution repo, listing each plugin's name, description, version, and `source` path. Users add the marketplace once (`copilot plugin marketplace add <org>/<repo>`), then install plugins from it individually. No enterprise licensing or admin-side config involved — this is the same self-service mechanism already used for this org's other Copilot CLI skills/agents.

## 5. Open Questions

- [ ] The output-token correction calls Anthropic's token-counting API from the `Stop` hook — what credentials does the hook use to authenticate that call (reuse the user's existing Claude Code auth, or a separate API key), and what's the fallback if the hook has no network access or the call fails (skip the correction and log the raw undercounted figure with a caveat, retry, or skip the log entry entirely)?
- [ ] Where does the local JSONL log live (path, one file per session vs. one running file), what's the retention/rotation policy, and does it need explicit write-safety for concurrent sessions appending at once (or are single-line appends safe as-is)?
- [ ] What mechanism does Copilot CLI expose for a persistent/custom statusline (plugin API, hook type, config file) — and does one exist yet? Needs research before implementation.
- [ ] Should `CLAUDE.md`'s repo-structure section be updated to match `phase2-kickoff.md`'s combined-repo revision, to avoid the two documents disagreeing?

---

## Status Legend

| Status | Meaning |
|--------|---------|
| Draft | Still being shaped; open questions may exist |
| Ready | Open questions resolved; safe to run `/plan-draft` |
| Implemented | All linked plan phases are Done |
| Superseded | Replaced by a newer spec — link it above |
