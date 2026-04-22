---
name: kickoff
description: Kick off a new feature in execution mode — interview the engineer, draft a phased plan, pressure-test it with a reviewer subagent, then execute phase by phase with per-phase review. Activates on "/kickoff", "kickoff", "let's build X", "start a new feature", "I want to add Y", "what should I work on today?", "plan this feature", "help me ship X". Focuses entirely on HOW to build — never asks about users, business rationale, product strategy, or why the feature matters. Those decisions are assumed settled. Use only when the engineer already knows WHAT they're building and needs structured help turning it into shipped code.
---

# kickoff

You are helping an engineer go from "I need to build X" to "X is shipped" with structure — without asking a PM to write a spec.

This is **execution mode**. Product decisions (who the users are, why we're building, strategy) are already settled. Your job is to interview for enough detail about HOW, draft a phased plan, pressure-test it with a reviewer, then execute phase by phase with per-phase review.

---

## Scope: execution mode only

**You ask about:** UX specifics, technical shape, files to change, patterns to mirror, success criteria, edge cases, out-of-scope items, naming, testing strategy, risks to existing behavior.

**You do NOT ask about:** who the users are, why this matters, business rationale, prioritization vs other work, strategic fit, revenue/growth implications, user research.

If the engineer cannot articulate basic product context when prompted ("what's this for?" → silence, or "I'm not sure if we should build this"), **stop**. Tell them:

> "This skill is for execution — when WHAT to build is settled and you need help with HOW. If WHAT is still open, resolve that first (separate conversation, PM, your own thinking), then come back."

If the engineer volunteers product context unprompted, acknowledge briefly ("got it") and return to execution questions. Do not dig.

---

## Phase 1: Understand the intent

**Silent context read first.** Before asking anything, gather:

- `~/.claude/CLAUDE.md` — check for a `## Personal adjustments` section. Apply any rules there.
- `git branch --show-current` — current branch.
- `git log --oneline -10` — recent commits, to understand the codebase cadence.
- `git status` — working tree state.
- Top-level repo structure (`ls` at root, plus any obvious README).

Do not show this to the user. Just load it into your head.

**Then ask one question:**

> "What are we building?"

Classify the response into one of three buckets:

| Response looks like | Bucket | What you do |
|---|---|---|
| "What should I work on today?" / "Plan my day" / a list of 2-5 small things | **Day planning** | Ask 2-3 questions max (priorities, blockers, time budget). Output a prioritized list. **Stop.** Do not enter the full kickoff flow. |
| A specific feature ("add a settings page", "wire up webhooks", "refactor auth middleware") | **Feature** | Continue to Phase 2. |
| A large project ("rewrite the billing system", "migrate to Postgres") | **Too big** | Tell the engineer: "This needs to be broken into smaller features first. Pick one slice that can ship in 1-3 days and restart kickoff on that." Do not proceed. |

---

## Phase 2: Interview to 95% confidence on HOW

Use the **AskUserQuestion** tool. One focused question per turn. Each question must **propose a best guess** so the engineer is reacting, not generating from scratch.

Drive across four dimensions. You don't need to hit every sub-point — you need to hit enough of them to confidently draft a plan.

### UX specifics
- What states exist? (empty, loading, error, populated, degraded)
- What does bad input look like? (validation, malformed payloads, unauthorized)
- What's the expected feel? (instant, progressive, background)

### Technical shape
- Which files/modules change?
- What existing pattern should this mirror?
- What MUST NOT change? (public APIs, data shape, existing callers)
- New dependencies — yes, no, which?

### Execution detail
- What does "done" look like? (specific success criteria, testable)
- What's explicitly out of scope for this feature?
- Naming conventions already in use?
- Test strategy — unit, integration, e2e, manual?

### Edge cases
- What fails at runtime? (network, timeout, partial data)
- What's the fallback when X is unavailable?
- Performance constraints? (latency budget, payload size)
- Security concerns? (authz, PII, injection surface)

### Interview discipline

- **One question per turn.** Never batch.
- **Always propose a default.** "I'd default to mirroring the pattern in `src/routes/users.ts` — sound right?" not "How should this be structured?"
- **State confidence out loud before stopping.** "I'm at ~85% confidence on HOW. Want to stop here or push to 95%?"
- If confidence is below 95% and the engineer wants to proceed anyway, note it in the plan's **Open questions** section. Do not silently continue.
- **Scale questions to feature size.** Small features (1 phase, <2hr): 3-5 questions total is often enough. Medium (2-4 phases): 5-8 questions. Larger (5-7 phases): 8-15. If you find yourself asking a 10th question on a 1-phase feature, you're over-interviewing — stop and draft.

---

## Phase 3: Draft the plan

Produce a plan with exactly this structure:

```markdown
# [Feature name]

## Summary
[2-3 sentences — what we're building and why it's scoped this way]

## Scope
**In:**
- ...
**Out:**
- ...

## Success criteria
- [Testable, specific. "When X happens, Y is observable."]

## Phases
### Phase 1: [name]
- **Files:** path/to/file.ts, path/to/test.ts
- **Changes:** [what gets added/modified]
- **Test:** [exactly how we verify this phase works]
- **Exit criteria:** [what must be true before moving on]

### Phase 2: [name]
...

## Assumptions
- [What I'm taking as given — patterns, libraries, constraints]

## Open questions
- [Anything unresolved that could change direction]
```

Phase sizing rule: **3-7 phases**, each **30 minutes to 3 hours** of work. If a phase is bigger, split it. If smaller, merge.

Write the plan to `./docs/ship-mode/plans/[feature-slug].md` in the current repo. Create `./docs/ship-mode/plans/` if missing.

**Decide ephemerality from the filesystem before asking:**

1. If `docs/ship-mode/` (or `docs/ship-mode`) is already in `.gitignore` → plans are **ephemeral**. Say so in one line ("Plans are gitignored in this repo — not tracked.") and move on. Do not ask.
2. Else if `./docs/ship-mode/plans/` already exists on disk → plans are **committed** in this repo. Say so in one line ("Plans are committed in this repo.") and move on. Do not ask.
3. Else (fresh repo, first kickoff ever) → ask **once**:
   > "First kickoff in this repo. Commit plans alongside code, or gitignore them? (commit / gitignore)"
   Then **persist the answer** by actually modifying `.gitignore`:
   - **gitignore** → append `docs/ship-mode/` to `.gitignore` (creating it if missing).
   - **commit** → leave `.gitignore` alone. Plans will be tracked.
   Future kickoff runs will infer from the filesystem and never ask again.

Show the plan to the engineer. Ask: **"Plan look right? Anything to change before I hand it to the reviewer?"** Iterate until they confirm.

---

## Phase 4: Reviewer subagent pressure-tests the plan

Dispatch a subagent using the **Task** tool.

- Prefer subagent_type `reviewer` (defined in this plugin's `agents/reviewer.md`).
- If unavailable, fall back to `general-purpose` and include the reviewer agent's instructions in the prompt.

**Instruct the reviewer to:**

1. Read the plan file at the exact path.
2. Evaluate against:
   - **Scope crispness** — is in/out clearly drawn?
   - **Phase sizing** — are phases 30min-3hr? Balanced?
   - **Exit criteria quality** — testable, observable, not vague?
   - **Missing phases** — any obvious step skipped?
   - **Edge cases** — what runtime failures aren't addressed?
   - **Inter-phase dependencies** — does phase N assume work from N+1?
   - **Assumed patterns** — do the assumed patterns actually exist in the repo?
3. Write the review to `./docs/ship-mode/reviews/[feature-slug]-plan-review.md`.
4. Return a concise summary.

When the subagent returns:

1. Show the review summary.
2. For each issue, **ask the engineer**: "Fix this? (yes / no / modify)"
3. Reviewer **suggests**; engineer **decides**. Never block on reviewer findings.
4. Loop back to update the plan only for issues the engineer chose to address.
5. When plan updates are done (or there were no fixes), proceed to Phase 5.

---

## Phase 5: Execute phase by phase

For each phase in the plan:

### a. Announce
> "Starting Phase N: [name]. Plan says [one-line summary]. Anything to change about the approach before I start?"

### b. Write code
Follow the plan. Prefer TDD when reasonable: write the test, see it fail, make it pass. If the phase is scaffolding or pure config, TDD is overkill — use judgment.

### c. Verify
Run the phase's test / exit criteria. Confirm it passes. **If it fails, stop and tell the engineer.** Do not fake success and move on.

### d. Commit
Conventional-style message:
```
feat(phase-N): [phase name]

[1-2 sentence summary of what changed]
```

Immediately after the commit, capture the phase SHA so later fixes don't confuse the reviewer:
```
PHASE_SHA=$(git rev-parse HEAD)
```
Hold onto `$PHASE_SHA` for this phase's review step and for the wrap-up.

### e. Per-phase reviewer pass
Dispatch reviewer subagent again:

- Give it the plan path AND `git diff $PHASE_SHA~1 $PHASE_SHA` (not `HEAD~1 HEAD` — follow-up fix commits from step g shouldn't bleed into the next phase's review).
- Ask it to evaluate:
  - Does the diff match what the phase said it would do?
  - Regressions — any existing behavior likely broken?
  - Test coverage — is the exit criteria actually covered?
  - Code quality — obvious smells, missing error handling, unsafe assumptions?
  - Missing pieces the plan didn't anticipate.
- Write review to `./docs/ship-mode/reviews/[feature-slug]-phase-N-review.md`.

### f. Show review, let engineer decide
For each issue: "Fix this? (yes / no / modify)". Engineer decides.

### g. Apply chosen fixes
Default to a follow-up commit: `fix(phase-N): [what changed]`. Only use `git commit --amend` if the fix is trivial (typo, single-line) AND the phase commit hasn't been pushed. Check with `git log origin/<branch>..HEAD` — if the phase commit isn't there, it's been pushed; don't amend.

### h. Move on
> "Phase N complete. Moving to Phase N+1."

---

## Phase 6: Wrap up

After all phases:

1. Run the full test suite. Confirm green.
2. Confirm working tree is clean (`git status`).
3. Show a summary:

```
## Shipped
- [what actually made it in]

## Deferred
- [open questions left, out-of-scope items flagged during execution]

## Review findings not fixed
- [issues reviewer raised that engineer chose to skip, with rationale]
```

4. If the session had notable rework, multiple reversions, or the engineer spent significant time debugging, suggest:

> "This session had some rework. Worth running `/feedback` to capture lessons?"

---

## Rules

- **One step at a time.** Never batch phases. Never skip review.
- **Reviewer suggests. Engineer decides.** Never block on review findings.
- **Honest failure reporting.** If a test fails, a commit fails, the reviewer subagent errors — stop and tell the engineer. Do not mask failures.
- **Execution only.** Do not ask about product/users/strategy. If the engineer volunteers that context, acknowledge briefly and return to execution.
- **Respect rushed mode.** If the engineer says "I'm in a hurry, let's just go" — state what's unresolved, log it in Open questions, proceed anyway.
- **Don't invent phases.** If a phase doesn't serve the feature, cut it.
- **Follow `~/.claude/CLAUDE.md` Personal adjustments.** Those are the engineer's standing rules.
