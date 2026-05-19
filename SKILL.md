---
name: cheapskate
description: Guides the agent to be economical with AI credits and premium requests. Load this skill when the user asks you to conserve credits, reduce premium request usage, or work in "cheapskate mode". Apply these principles when you notice the task could be solved with fewer turns or cheaper tools.
user-invocable: true
---

# Cheapskate Skill — Economy Mode

**ON by default.** No explicit activation needed. First time in a session, announce once:
> "Cheapskate mode active. I'll minimize tool calls, batch operations, and use MC/QQ for input. Say 'cheapskate off' to disable."

Toggle: **"cheapskate off"** / **"cheapskate on"**. After the announcement, operate silently.

Every model response costs one premium request. Subagent calls do **not** cost extra (verified) — use them freely when they're the right tool.

---

## Model Tier Awareness

> Table is static — verify at: https://docs.github.com/en/copilot/managing-copilot/monitoring-usage-and-spending/about-premium-requests

| Multiplier | Models |
|------------|--------|
| **x1** | Claude Sonnet 4.5/4.6, GPT-4o, GPT-4o-mini, Gemini 1.5 Pro |
| **x5–x10** | Claude Opus 4, o1, Gemini 2.0 Pro |
| **x15–x20** | o3, o3-mini (high), Claude Opus 3 |

- **x1**: no action needed.
- **>x1**: warn at session start: _"You're on [model] (x[N] rate). Minimizing calls."_
- **Subagents are capped at the parent model's tier** — the platform enforces this.

---

## Tool Cost Hierarchy

| Tier | Tools |
|------|-------|
| **Free** | `read_file`, `list_dir`, `file_search`, `grep_search`, `get_errors`, `memory`, `run_in_terminal` |
| **Moderate** | `semantic_search` (embedding call) |
| **No extra premium request** | `runSubagent` (use freely when appropriate) |

**Rules:**
- `grep_search` before `semantic_search` — only use semantic when you don't know what text to look for.
- `file_search` for locating files by name/pattern.
- `read_file` directly if you already know the path.
- `runSubagent` for deep multi-file exploration or tasks that would clutter the main conversation — don't avoid them just to save latency if they're the right tool.

---

## Do More Per Turn

**Parallel:** Call independent reads (`read_file`, `list_dir`, `grep_search`) in parallel. Use `multi_replace_string_in_file` over sequential single-file edits. Front-load all context gathering — decide what you need, read it all at once.

**Scope reads:** Use `startLine`/`endLine` — don't read 2000 lines to find a 10-line function.

**Stop early:** 2–3 converging results = enough context. Trust findings; don't re-search to confirm. Don't read a file you just wrote — trust the tool.

**Use session memory:** Write computed/discovered facts to session memory rather than rediscovering them next turn.

**Anti-patterns to avoid:** cascade searching, confirmation reads, speculative subagents, sequential small reads, redundant re-reads.

---

## MC/QQ — Eliminate Typing Turns

> **MC/QQ** = Multiple Choice / Quick Question — the `vscode_askQuestions` tool. Renders clickable buttons in the chat UI. **Costs nothing** (no model inference).

**Never ask a yes/no or choice question in plain text.** Always use `vscode_askQuestions` with options — the human clicks instead of types, eliminating ambiguity and saving a follow-up turn.

**End-of-response rule:** Close every response with an MC/QQ — unless it's a pure factual answer with no logical follow-up, or a question was already asked inline.

### Collaborative vs. Autonomous Flow

Adapt MC/QQ frequency to the session mode:

| Mode | Signals | MC/QQ frequency |
|------|---------|----------------|
| **Collaborative** | Short single requests; user commenting each step; no broad pre-approval | Ask freely — mid-response for choices, end-of-response for next step |
| **Autonomous** | "just do it", "handle it all", Pulse/Dream cycle, "stop asking me" | Only at end of full task or hard irreversible decision points |

When in doubt → **collaborative**. If uncertain mid-task, ask once: _"Continue autonomously or check in after each step?"_

---

## Summary Checklist (Before Each Tool Call)

1. Is there a cheaper tool that gets the same result?
2. Can I batch this with other reads/writes?
3. Do I already have this from earlier in the conversation?
4. Am I reading more than I need to?
5. Would a subagent handle this better than doing it inline?
