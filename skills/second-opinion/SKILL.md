---
name: second-opinion
description: Prepare a plan or diff for review by an external model (Claude.ai web, Codex, GPT, another LLM). Activates on "/second-opinion", "get a second opinion", "review with codex", "external review", "fresh eyes on this plan". Formats the artifact plus a skeptical-reviewer prompt and copies both to the clipboard so the engineer can paste into another tool. Used when genuinely different model reasoning matters — not just what the same Claude would say from another angle.
---

# second-opinion

## When to use this

The built-in `reviewer` subagent shares context with the current session — the same prompts, the same file reads, the same conversation history. That shared context biases it toward what's already been decided: it's great at catching slop inside the chosen frame, but it rarely questions the frame itself. Cross-model review gets genuinely different takes — different model weights, different blind spots, different frames. Use this skill when the stakes justify that extra friction: launch-critical features, architecture decisions with long-lived consequences, anything high-consequence where being wrong is expensive. Not for every kickoff.

---

## Two modes

### Plan review mode

Activates when the engineer asks for a second opinion and a plan file exists in `./docs/ship-mode/plans/` in the current repo.

- If exactly one plan file exists → use it.
- If multiple plan files exist → use the **AskUserQuestion** tool to let the engineer pick one.
- If no plan file exists → tell the engineer "No plan found in `./docs/ship-mode/plans/`. Run `/kickoff` first, or use code/diff mode instead." and stop.

### Code/diff review mode

Activates when the engineer specifies a diff, commit SHA, or branch range. Translate common phrasings into `git diff` invocations:

| Phrasing | Command |
|---|---|
| "`/second-opinion` on the last N commits" | `git diff HEAD~N HEAD` |
| "`/second-opinion` on main" | `git diff main...HEAD` |
| "`/second-opinion` on `<sha>`" | `git show <sha>` |
| "`/second-opinion` on `<branch>`" | `git diff <branch>...HEAD` |

If the diff is empty, tell the engineer and stop.

---

## Output format

Produce a **single text block** the engineer can paste elsewhere. Structure:

```
You are a skeptical staff engineer reviewing a [plan | code diff]. The author wants honest critique — not validation. Find load-bearing assumptions that may not hold, architectural alternatives the author didn't consider, missing phases or edge cases or failure modes, and premise issues (is this the right approach at all?). Be direct. Be specific. Do not soften findings. Categorize every finding as CRITICAL (would cause bugs or rework), MODERATE (worth fixing), or NIT (minor). End with a verdict of green, yellow, or red.

---

[the actual plan content, or the actual diff content]

---
```

Use the word "plan" or "code diff" in the first line depending on mode. The separators are literal three-dash lines.

---

## Clipboard copy

After producing the output block, copy it to the system clipboard by detecting the platform:

- **macOS** → pipe into `pbcopy`.
- **Linux** → try `xclip -selection clipboard` first. If `xclip` is missing, fall back to `xsel -ib`.
- **WSL** → pipe into `clip.exe`.
- **None available** → skip the copy step. Print the block to stdout and tell the engineer "No clipboard tool found — copy the block above manually."

Detect in this order: `uname` → if `Darwin`, use `pbcopy`. If `Linux` and `/proc/version` mentions `microsoft`, use `clip.exe`. Otherwise on Linux, probe `xclip` then `xsel`.

---

## After copying

Tell the engineer, verbatim:

> Copied to clipboard. Paste into Claude.ai, Codex, or another model. When their review comes back, you can paste it here and I'll help reconcile it with the built-in reviewer's findings.

If clipboard copy was skipped (no tool available), replace "Copied to clipboard." with "No clipboard tool available — block printed above."

---

## Rules

- **Read-only.** Never modify the plan file or the diff source. This skill only reads, formats, and copies.
- **Do not run the review yourself.** The built-in `reviewer` subagent handles in-session review. This skill exists only to format content for an **external** model. Do not write a review, do not summarize the content, do not give your own take.
- **Truncate to 8000 characters if needed.** If the full output block (instruction + content + separators) exceeds 8000 characters, truncate the pasted plan/diff content (not the instruction) to fit, and warn the engineer: "Content truncated at ~Nchars. External reviewer is seeing a partial view."
- **Secret scan before copying.** If the plan or diff contains strings that look like secrets — API keys (e.g., long base64/hex runs preceded by `key=`, `token=`, `secret=`, `AWS_`, `ghp_`, `sk-`), `-----BEGIN PRIVATE KEY-----` blocks, or `password:` / `passwd=` followed by non-placeholder values — **stop before copying**. List the suspicious lines and let the engineer decide whether to proceed, sanitize, or abort.
- **Single pass.** Format, copy, tell the engineer, stop. Do not iterate, do not offer refinements, do not re-run.
