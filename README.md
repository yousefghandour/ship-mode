# ship-mode

A Claude Code plugin for engineers who already know **what** they're building and want help shipping it well — no PM required.

---

## What it is

Three skills and one subagent, bundled as a Claude Code plugin. You talk to Claude Code like you normally would; the skills activate on the phrases below.

- **`kickoff`** — interview → phased plan → reviewer pass → execute phase by phase with per-phase review
- **`coach`** — reads your local session logs and suggests specific improvements to how you prompt
- **`feedback`** — reflects on a session (or the last few days) and asks "what could you have told me upfront?"
- **`second-opinion`** — formats a plan or diff for review by an external model (Claude.ai web, Codex, another LLM)
- **`reviewer`** subagent — skeptical staff engineer that pressure-tests plans and diffs for `kickoff`

All three skills write any accepted adjustments to your personal `~/.claude/CLAUDE.md`. Nothing is shared with the team.

---

## Why

Execution-mode feature work has a gap: you know **what** to build (the product decisions are settled), but there's still a lot of HOW to nail down — file paths, patterns to mirror, edge cases, phasing, what must NOT change — and writing a full spec by hand is tedious overhead.

`kickoff` fills that gap: it interviews you to ~95% confidence on HOW, drafts a phased plan, runs a skeptical reviewer against it, then executes phase by phase with a reviewer pass per phase. **The reviewer suggests; you decide.**

`coach` and `feedback` are the feedback loops: they watch how you prompt and how your sessions unfold, and help you get better over time — again, all personal to you.

---

## The three skills

### `kickoff`

> Activation: `/kickoff`, "kickoff", "let's build X", "start a new feature", "I want to add Y", "what should I work on today?", "plan this feature", "help me ship X"

What it does:
1. **Reads your context silently** — `~/.claude/CLAUDE.md` personal adjustments, current branch, recent git log, working tree.
2. **Asks one question** — "What are we building?" — then classifies: day-planning / feature / too-big.
3. **Interviews to 95% confidence on HOW** — UX specifics, technical shape, success criteria, edge cases. Scales question count to feature size (3-5 for small, 5-8 for medium, 8-15 for larger).
4. **Drafts a phased plan** at `./docs/ship-mode/plans/[slug].md` — decides commit vs gitignore from the filesystem (asks only on first kickoff in a repo).
5. **Dispatches the `reviewer` subagent** against the plan. Shows findings. For each issue: "Fix this? (yes / no / modify)."
6. **Executes phase by phase** — write code, verify tests, commit, capture phase SHA, reviewer pass on that phase's diff, apply any chosen fixes (default to follow-up commits, not amend).
7. **Wraps up** — full test suite, summary of what shipped / deferred / skipped, suggests `/feedback` if the session had rework.

Example:
> `let's build a settings page for user notifications`
> kickoff activates, asks "What are we building?", interviews UX + technical shape, drafts 4-phase plan, reviewer flags one missing phase and vague exit criteria, you fix those, then execute.

### `coach`

> Activation: `/coach`, "coach me", "coach my prompts", "coach me on my last N prompts", "how have my prompts been", "review my prompting"
>
> Also auto-offers every ~10 original prompts in a session.

What it does:
1. Reads the current session's `.jsonl` at `~/.claude/projects/`.
2. Filters to **original prompts** only — skips yes/no replies, clarifications, short follow-ups.
3. Scores each against: file paths named? success criteria stated? scope drawn? context provided? actionable without follow-up?
4. Surfaces the top 3 **patterns** of weakness (not individual prompts). Shows before/after rewrites drawn from your actual logs.
5. Proposes 1-3 standing adjustments. For each: **accept / reject / modify / defer**. Accepted adjustments append to `~/.claude/CLAUDE.md` under `## Personal adjustments`:
   ```
   - [2026-04-21, coach] When asking for a refactor, list what must NOT change.
   ```

Zero adjustments is a valid outcome — if your prompts were strong, it says so and stops.

Example:
> `/coach`
> "Looked at your last 12 original prompts. Top pattern: starting refactors without naming what must NOT change (saw this in 4 of them). Proposed adjustment: [...]. accept / reject / modify / defer?"

### `feedback`

> Activation: `/feedback`, "what should I have done differently", "post-mortem this session", "reflect on this session", "what went wrong", "lessons from this session"
>
> Also auto-offers after sessions with notable rework, and every ~3 days for a multi-day reflection.

What it does:
1. **Per-session mode** — walks the current session's `.jsonl`, finds the 1-3 most costly moments (time that didn't need to go that way), and asks two questions for each:
   - What could you have said upfront to prevent this?
   - What could Claude have done better?
2. **Multi-day mode** — looks at `.jsonl` files from the last 3 days, surfaces recurring patterns that appeared in 2+ sessions.
3. Distills into 1-3 **standing, testable, one-sentence rules**. For each: **accept / reject / modify / defer**. Accepted lessons append to `~/.claude/CLAUDE.md`:
   ```
   - [2026-04-21, feedback] Before writing migrations, check what the downstream consumers read from the table.
   ```

Zero lessons is a valid outcome — if the session went smoothly, it says so and stops.

### `second-opinion`

> Activation: `/second-opinion`, "get a second opinion", "review with codex", "external review", "fresh eyes on this plan"

What it does:
1. **Detects the mode** — plan review (if a plan exists in `./docs/ship-mode/plans/`) or code/diff review (if you name a diff, SHA, or branch range like "the last 3 commits" or "on main").
2. **Builds a single paste-ready block** — a skeptical-staff-engineer instruction paragraph, then a `---` separator, then the actual plan or diff content, then another `---`.
3. **Scans for secrets before copying** — if anything looks like an API key, token, private key, or password, it stops and shows you the suspicious lines so you can sanitize or abort.
4. **Copies to the system clipboard** — `pbcopy` on macOS, `xclip` / `xsel` on Linux, `clip.exe` on WSL. If no clipboard tool is available, it prints the block and tells you to copy manually.
5. **Truncates at 8000 characters** if the block is too large, and warns you that the external reviewer is seeing a partial view.
6. **Tells you what to do next** — paste into Claude.ai, Codex, or another model; then come back with the review and ship-mode helps reconcile it with the built-in reviewer's findings.

Example:
> `/second-opinion on the settings-page plan`
> Skill finds `./docs/ship-mode/plans/settings-page.md`, formats the block, scans clean for secrets, copies to clipboard. "Copied. Paste into Claude.ai or Codex, then bring the review back."

Use this when stakes justify cross-model review — launch-critical features, architecture decisions, anything where being wrong is expensive. Not for every kickoff.

---

## The reviewer subagent

`kickoff` dispatches a dedicated `reviewer` subagent for plan review and per-phase review. The reviewer:

- Categorizes every issue as **CRITICAL**, **MODERATE**, or **NIT**.
- Returns a verdict: **green** (ship it), **yellow** (address moderates), **red** (critical issues present).
- Returns a structured summary: path, verdict, counts, top issue — parseable by `kickoff`.
- **Suggests; does not block.** The engineer decides what to fix.

---

## Installation

ship-mode ships a `.claude-plugin/marketplace.json` that exposes the plugin as part of a single-plugin marketplace also named `ship-mode`. The install syntax is `<plugin-name>@<marketplace-name>` — both are `ship-mode`, hence `ship-mode@ship-mode`.

### From a local clone (recommended while iterating)

```
git clone https://github.com/yousefghandour/ship-mode.git ~/code/ship-mode
```

In Claude Code:
```
/plugin marketplace add ~/code/ship-mode
/plugin install ship-mode@ship-mode
```

### From GitHub (once published)

```
/plugin marketplace add yousefghandour/ship-mode
/plugin install ship-mode@ship-mode
```

Verify install:
```
/plugin
```
You should see `ship-mode` in the list.

---

## Philosophy

- **Execution mode only.** ship-mode assumes WHAT to build is settled. It does not ask about users, business rationale, or strategy. If those are open, resolve them first.
- **Reviewer suggests. Engineer decides.** The reviewer subagent pressure-tests plans and diffs and flags concerns, but it never blocks execution. You choose what to fix.
- **Personal, not team-shared.** `coach` and `feedback` write accepted rules to your own `~/.claude/CLAUDE.md`. Nothing is pushed to shared team infra. Your rough edges stay yours.
- **Self-contained.** ship-mode does not depend on Superpowers, prompt-coach, prompt-improver, or any other plugin. It is designed to be installable on its own.
- **No invented problems.** `coach` and `feedback` propose zero adjustments when there's nothing to improve. They do not manufacture lessons to justify their existence.

---

## File layout

```
ship-mode/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── coach/SKILL.md
│   ├── feedback/SKILL.md
│   ├── kickoff/SKILL.md
│   └── second-opinion/SKILL.md
├── agents/
│   └── reviewer.md
├── README.md
├── LICENSE
└── .gitignore
```

Plans and reviews written by `kickoff` land in `./docs/ship-mode/plans/` and `./docs/ship-mode/reviews/` in your target repo — not here.

---

## License

MIT. See `LICENSE`.
