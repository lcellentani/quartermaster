# Recon: `JuliusBrussee/caveman`

Repo: https://github.com/JuliusBrussee/caveman
Purpose (from README): a portable "skill" that makes AI coding agents (Claude Code, Codex, Gemini CLI, Cursor, Windsurf, Cline, Copilot, opencode, OpenClaw, 30+ others) respond in a compressed, filler-free style — measured **65% fewer output tokens**, technical accuracy unchanged. It does **not** touch input tokens or reasoning; it constrains what the agent *says*.

This is not really a "framework" in the engine-architecture sense you and I usually mean the word — there's no runtime, no game loop. It's a **prompt-engineering + distribution toolkit**: one behavioral spec (the SKILL.md), one lightweight session-lifecycle mechanism (hooks + flag file), and one installer that fans that spec out to 30+ different agent ecosystems, each with its own plugin format. That's the actual thing worth studying.

---

## 1. Footprint

- ~2.0 MB total, MIT licensed, Node ≥18.
- **21 JS files / 4,312 LOC** in `bin/` + `src/` (the installer + hooks + tools — the only "real" application code).
- **13 Python files** (`evals/`, `benchmarks/`, some tests) — used for measurement, not runtime.
- **44 Markdown files** — this is the tell: the product's actual logic mostly lives in Markdown (SKILL.md prompt bodies + subagent specs), not in code. Code exists to *install and activate* the Markdown, not to implement behavior.
- 32 test files (Node's built-in test runner + `pytest`), covering the installer, hooks, and compression safety.
- No runtime dependencies of note — `package.json` only declares dev/test tooling. No network calls after install (explicitly documented as a privacy guarantee).

## 2. Directory map (annotated)

```
caveman/
├── skills/                  # SOURCE OF TRUTH — one dir per skill, {SKILL.md, README.md}
│   ├── caveman/              # the core behavior spec (see §3)
│   ├── caveman-commit/       # independent skill: commit message style
│   ├── caveman-review/       # independent skill: PR review style
│   ├── caveman-compress/     # sub-skill: rewrites OTHER memory files into the compressed style
│   ├── caveman-help/         # one-shot reference card (not persistent mode)
│   └── cavecrew/             # decision guide for delegating to the subagents below
│
├── agents/                  # 3 caveman-flavored SUBAGENTS (see §5)
│   ├── cavecrew-investigator.md   # read-only locator, haiku model, refuses to edit
│   ├── cavecrew-builder.md        # surgical 1-2 file editor, refuses 3+ file scope
│   └── cavecrew-reviewer.md       # diff reviewer, one-line findings
│
├── src/
│   ├── hooks/                # Claude Code lifecycle hooks (see §4) — the only "always-on" mechanism
│   ├── rules/                # always-on rule body consumed by per-repo IDE rule writers
│   ├── tools/caveman-init.js # writes per-repo rule files (.cursor/rules, .windsurf/rules, etc.)
│   └── mcp-servers/caveman-shrink/  # MCP middleware: wraps any MCP server, compresses ITS tool descriptions
│
├── bin/install.js           # single unified installer — PROVIDERS[] array, one entry per agent (see §6)
├── bin/lib/settings.js      # JSONC-tolerant settings.json reader/writer (Claude Code config is JSON-with-comments)
├── bin/lib/openclaw.js      # per-agent installer helper example (frontmatter merge, idempotent marker blocks)
│
├── plugins/caveman/         # CI-mirrored DISTRIBUTION copy for the Claude Code plugin loader
│                             #   (plugin loader requires skills/agents at a specific path; CI syncs skills/ → here)
├── dist/caveman.skill       # gitignored ZIP build artifact, rebuilt by CI
│
├── commands/                # Codex/Gemini TOML command stubs (root, for plugin auto-discovery)
├── evals/                   # 3-arm eval harness: baseline vs terse-instruction vs skill (see §7)
├── benchmarks/               # real Claude-API token counts backing the README's 65% claim
└── tests/                    # installer + hook + compression-safety tests (Node + pytest)
```

Two structural decisions stand out:

1. **Single source, many mirrors, one sync job.** `skills/` is canonical; `plugins/caveman/` is a CI-generated mirror because the Claude Code plugin loader requires a specific path layout. A GitHub Action (`sync-skill.yml`) copies on every push and commits back with `[skip ci]`. They were explicit that this used to also mirror into `.cursor/`, `.windsurf/`, `.clinerules/` etc. and **removed** that — those dotdirs were dead weight, never actually read by the installer.
2. **CLAUDE.md as an internal engineering doc, not a starter prompt.** Their CLAUDE.md (24K) isn't "tell the agent about the repo" boilerplate — it's a precise ownership map ("edit only these files," "never edit synced copies," symlink-attack notes on flag writes, JSONC gotchas). It reads like a senior engineer's onboarding doc for future Claude Code sessions specifically, which is a good model for whatever framework we design for you.

## 3. Core mechanism: the SKILL.md as a compiler target

`skills/caveman/SKILL.md` (78 lines) is the actual "engine." Frontmatter declares trigger phrases; body is the compressed rule set:

- **Persistence contract**: "ACTIVE EVERY RESPONSE… Off only: 'stop caveman' / 'normal mode'." No soft decay.
- **Concrete drop-list**: articles, filler words, pleasantries, hedging — but an explicit **do-not-touch list**: code blocks, exact error strings, CLI commands, invented abbreviations are *banned* (their own measurement showed invented shorthand doesn't save tokens — tokenizer splits it same as the full word — so it's clarity loss for zero gain).
- **Six intensity levels** (`lite → ultra → wenyan-ultra`), each defined by a diff against the previous, anchored with two paired before/after examples per level. This few-shot-per-level pattern is doing a lot of the calibration work — it's not just telling the model "be terser," it's showing the exact register at each step.
- **An escape hatch** ("Auto-Clarity"): security warnings, irreversible actions, and multi-step sequences where compression itself creates ambiguity force a drop back to normal prose, with a worked example (a `DROP TABLE` warning). This is the detail I'd flag as most transferable — any "always-on style constraint" framework needs an explicit, example-anchored override for the cases where the constraint would hurt.

## 4. Distribution inside Claude Code: hooks + flag file, not memory

For Claude Code specifically (the agent you'd care most about), activation doesn't rely on the model remembering an instruction across a long session — it's enforced statefully:

```
SessionStart hook ──writes "full"──▶ $CLAUDE_CONFIG_DIR/.caveman-active ◀──writes mode── UserPromptSubmit hook
                                                       │
                                                    reads
                                                       ▼
                                              caveman-statusline.sh → "[CAVEMAN] ⛏ 12.4k"
```

- `caveman-activate.js` (SessionStart): writes the mode flag, emits the ruleset as **hidden stdout** (Claude Code injects SessionStart stdout as system context, invisible to the user — worth knowing this exists), checks for statusline config and offers to set it up.
- `caveman-mode-tracker.js` (UserPromptSubmit): parses `/caveman <level>` and natural-language phrases ("talk like caveman" / "normal mode") to flip the flag; also does small **per-turn reinforcement** — a short reminder injected when the mode is active, specifically because other mid-conversation instructions (other plugins, other skills) compete for the model's attention and can erode a session-start-only instruction over a long conversation.
- Flag writes go through a hardened `safeWriteFlag()` — refuses if the flag path or its parent dir is a symlink, opens with `O_NOFOLLOW`, atomic temp+rename, `0600` perms. This exists specifically to block a local-attacker symlink-clobber against a predictable, user-writable path. Small detail, but it's the kind of thing worth stealing wholesale rather than reinventing.
- Everything **silent-fails** on filesystem errors — a hook must never block session start.

## 5. Subagent delegation pattern (`cavecrew`)

Three narrowly-scoped subagents, each with a hard-typed output contract so the *main thread* only ever ingests compressed results:

| Subagent | Model | Scope | Refuses |
|---|---|---|---|
| `cavecrew-investigator` | haiku | read-only locate (`Grep`/`Glob`/`Read`/`Bash`), outputs `path:line — symbol — note` | any fix or design suggestion |
| `cavecrew-builder` | — | surgical 1-2 file edit | 3+ file scope ("too-big.") |
| `cavecrew-reviewer` | haiku | diff/file review, one-line findings w/ severity emoji | — |

`skills/cavecrew/SKILL.md` is a decision table for the *orchestrating* agent — "if you'd want the output in 1/3 the tokens, use cavecrew; if you'd want prose, use vanilla Explore." The stated rationale: subagent tool results get injected into the main context **verbatim**, so a 2k-token vanilla `Explore` costs 2k tokens of main-context budget on every delegation, and across ~20 delegations in a long session that's the difference between finishing the task and hitting context exhaustion. This is a genuinely useful pattern for any long-running agentic workflow, independent of the caveman-speak angle.

## 6. Multi-agent portability

`bin/install.js` holds one `PROVIDERS` array — one declarative entry per agent:

```js
{ id: 'cursor', label: 'Cursor', mech: 'npx skills add (cursor)',
  detect: 'vscode-ext:cline', profile: 'cline' }
```

Each entry: `id`, `label`, `mech` (how it installs), `detect` (a small clause DSL — `command:x||dir:$HOME/y`), optional `profile` (a slug into the separate `vercel-labs/skills` registry project), optional `soft: true` for best-effort config-dir-only detection. Adding a new agent is: append one row, run `--list` to confirm rendering, verify the profile slug exists upstream. Distribution mechanism varies a lot per agent — real plugin (Claude Code, Codex), extension (Gemini), native plugin (opencode), workspace-file injection (OpenClaw), or just a written rule file for anything without a hook system (Cursor, Windsurf, Cline, Copilot, ~15 others via `npx skills add`).

`caveman-compress` is a distinct, smaller idea worth separating out mentally: it's a sub-skill that rewrites *other* memory files (like a `CLAUDE.md`) into the compressed style, on disk, with a `.original.md` backup and validation that headings/code/URLs/paths survive. That's a one-time, persistent input-token saving, as opposed to the main skill's per-response output-token saving — different axis, same philosophy.

## 7. Measurement discipline

Two separate harnesses, and they're careful about what they compare:

- `evals/`: three-arm — `__baseline__` (no system prompt), `__terse__` (just "Answer concisely."), and `<skill>` (terse instruction + full SKILL.md). They explicitly call out that comparing skill-vs-baseline would be "cheating" — it'd conflate the skill with generic terseness. The honest number is skill-vs-terse.
- `benchmarks/`: real Claude API calls, raw token counts, committed as JSON, README table generated from those files — not hand-typed.
- A dedicated `docs/HONEST-NUMBERS.md` caveats their own headline number: the skill only cuts output tokens, adds ~1-1.5k input tokens per turn itself, and can go net-negative on already-terse workloads.

That combination (adversarial baseline design + a document that argues against your own marketing number) is unusually disciplined for a project whose README also fully commits to a bit ("why use many token when few do trick").

---

## What this suggests for *your* framework

Not a design yet — just what the recon surfaces as directly reusable structure once we start Phase 2 on your machine:

1. **Separate the "spec" from the "delivery mechanism."** Their actual behavioral logic is a handful of Markdown files; everything else is plumbing to get that Markdown loaded into different tools at the right time. If your framework is about enforcing *your* coding conventions/review style/whatever inside Claude Code, the SKILL.md pattern (frontmatter trigger + rules + intensity-style few-shot examples + an explicit override clause) is a strong, proven starting template — and it's Claude Code-native, no separate infra needed.
2. **The hook + flag-file + statusline trio** is the piece specific to Claude Code persistence, and it's small enough to fully read in one sitting (`src/hooks/`, ~4 files). If you want something that's *always on* for your own sessions rather than re-invoked every time, this is the exact mechanism, symlink-safety included.
3. **`cavecrew`'s subagent-with-typed-output-contract pattern** is probably the most valuable piece independent of the "caveman" theme — it's a general answer to "how do I keep long agentic sessions from drowning main context in verbose tool results," which is a real problem regardless of whether compression-as-a-bit is your thing.
4. Given you're doing this for Frostbite-adjacent, C++-heavy, engine-architecture work — none of `caveman`'s actual *content* (the compression rules) is relevant to you. What's transferable is the **shape**: source-of-truth skill files + lifecycle hooks + measurement harness that keeps you honest about whether it's actually helping.

**Open questions for Phase 2** (once we're on your machine with Claude Code): do you want something that's Claude-Code-specific (simplest, reuses their hook pattern almost directly) or portable across whatever else you use day to day? And is the target behavior a *response style* constraint like theirs, or something closer to enforcing your engine's actual coding conventions/architecture rules — because that changes whether the SKILL.md pattern is the right shape at all, or whether you want something closer to their `cavecrew` subagent-contract idea instead.
