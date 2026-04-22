---
name: feedback
description: Reflect on the current session (or past few days) and answer "what could I have told you upfront to avoid the mistakes you made?" Activates on "/feedback", "what should I have done differently", "post-mortem this session", "reflect on this session", "what went wrong", "lessons from this session". Also suggests itself automatically after long sessions with notable rework, and offers a multi-day reflection roughly every ~3 days of active Claude Code use. Distills real friction into 1-3 concrete, testable, transferable rules and — with the engineer's approval — appends them to their personal `~/.claude/CLAUDE.md`. Personal to the engineer; nothing is shared with the team.
---

# feedback

You reflect on what actually happened in this session (or across the last few days) and surface transferable lessons. You answer one question honestly: **"What could the engineer have told me upfront to avoid the mistakes I made — and what could I have done better?"**

This skill is **personal**. Accepted lessons append to `~/.claude/CLAUDE.md`, which lives in the engineer's home directory and is never committed.

---

## When this skill runs

Three triggers:

1. **Manual invocation.** The engineer says `/feedback`, "what should I have done differently", "post-mortem this session", or similar → run the per-session reflection immediately.

2. **End-of-session suggestion.** At the end of a session that had **notable friction** — phase reversions, "actually let's do X instead" mid-flight, debugging consuming >50% of session time, multiple rollbacks — proactively offer:
   > "This session had some rework. Worth running `/feedback` to capture lessons? (yes / not now)"
   If the session went smoothly, do not offer.

3. **Multi-day reflection.** Roughly every **3 days of active Claude Code use** (check timestamps on `~/.claude/projects/*.jsonl` files — 3 days since the last `feedback` entry in `~/.claude/CLAUDE.md`, or no prior entry but 3+ days of session logs), offer:
   > "It's been a few days of Claude Code work. Want a reflection across the last 3 days? (yes / not now)"
   Run multi-day mode if they accept.

---

## Per-session reflection

### 1. Read the current session

Locate the most recent `.jsonl` at `~/.claude/projects/<cwd-encoded>/`. Walk it start-to-finish to reconstruct:

- The initial prompt and stated intent.
- Clarifications and context added along the way.
- **Points where direction changed** — where the engineer said "actually…", "wait, let me…", "let's back up", or rolled back a change.
- **Rework** — commits reverted, files rewritten after initial implementation, tests that had to be fixed after first pass.
- The final state (what shipped, what's left open).

### 2. Identify the 1-3 most costly moments

Not every bump is a lesson. Focus on the moments where **time went that didn't need to**. A 5-minute detour isn't a lesson. A 45-minute wrong-direction is.

### 3. For each costly moment, answer two questions

- **What could the ENGINEER have said upfront to prevent this?** Be specific. What piece of information, what constraint, what framing would have changed the trajectory? Not "more context" — "the fact that this table is append-only and we can't reorder rows."
- **What could CLAUDE have done better?** An assumption Claude made that should have been checked. A question Claude should have asked before writing. A pattern Claude missed in the existing code. Be honest — do not blame the prompt for Claude's own missteps.

### 4. Distill into 1-3 transferable lessons

Each lesson must be:
- **A standing rule.** Applies to future sessions, not just this one.
- **Testable.** You could tell from a future prompt whether it was followed.
- **One sentence.**
- **Specific.**

Good example:
> "When asking for a refactor, explicitly list what must NOT change."

Bad example:
> "Be clearer."

If there's nothing to extract (smooth session, real friction was unavoidable), propose **zero** lessons and say so.

---

## Multi-day reflection mode

When triggered (or explicitly requested: "reflect on the past 3 days"):

### 1. Read the past 3 days of session logs

Walk every `.jsonl` in `~/.claude/projects/` modified in the last 3 days. Reconstruct each session at the level of "what was the goal, where did friction happen, what did it cost."

### 2. Look for recurring patterns

One bad moment in one session is noise. The **same kind of friction appearing in 2+ sessions** is signal. Examples of recurring patterns worth flagging:
- Prompts kicking off coding before the target files were identified → same confusion each time.
- Tests written after implementation and finding the implementation was wrong.
- Refactors that broke unstated invariants repeatedly.

### 3. Produce 1-3 lessons across sessions

Same format as per-session. Same bar for quality. Same "zero is valid" rule.

---

## Approval flow

For each proposed lesson, use the **AskUserQuestion** tool with exactly four options:

- **accept** — append to `~/.claude/CLAUDE.md`
- **reject** — discard, do not resurface this session
- **modify** — engineer rewrites the lesson, then append the modified version
- **defer** — drop for now, may resurface later

For each **accepted** lesson:

1. Open `~/.claude/CLAUDE.md`.
2. Find the `## Personal adjustments` section. Create it at the end of the file if missing.
3. Append a line in this exact format:
   ```
   - [YYYY-MM-DD, feedback] [lesson text]
   ```
   Use today's date. Example:
   ```
   - [2026-04-21, feedback] Before writing migrations, check what the downstream consumers read from the table.
   ```

4. **Never commit `~/.claude/CLAUDE.md`.** It is personal.

Before proposing, read the existing Personal adjustments section and skip anything that duplicates an existing rule.

---

## Rules

- **Keep output under 300 words.** The engineer just finished a session — they want signal, not prose.
- **Be direct. No padding with positivity.** If the session went well, say one sentence and stop. Do not manufacture lessons.
- **Honest about Claude's own mistakes.** If Claude made a bad assumption the prompt couldn't reasonably have prevented, own it in the reflection. Do not blame the prompt for Claude's shortfalls.
- **Never expose specific file contents, credentials, or sensitive jargon** when summarizing costly moments. Generalize: "a config file in the auth module" not literal paths or values that could leak.
- **Zero lessons is a valid answer.** If the session was smooth or friction was genuinely unavoidable (unknown third-party bug, flaky infra), say so. Do not invent problems.
- **Respect "not now."** If the engineer declines a triggered suggestion, drop it. Do not nag.
- **Scope to what the engineer asked.** Per-session means this session. Multi-day means the last 3 days. Do not expand silently.
- **Read-only on project files.** This skill only reads session logs and writes to `~/.claude/CLAUDE.md`.
