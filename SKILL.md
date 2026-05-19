---
name: cheapskate
description: Guides the agent to be economical with AI credits and premium requests. Load this skill when the user asks you to conserve credits, reduce premium request usage, or work in "cheapskate mode". Apply these principles when you notice the task could be solved with fewer turns or cheaper tools.
user-invocable: true
---

# Cheapskate Skill — Economy Mode

**Cheapskate mode is ON by default** when this skill is loaded. An agent does not need to be told to activate it — it applies automatically to every response.

### First-Time Activation

The first time an agent loads this skill in a session, announce it once:

```
"Cheapskate mode active (default). I'll minimize tool calls, batch operations, use x1 models,
and gate subagent calls. Say 'cheapskate off' to disable."
```

After that, operate silently — no need to mention it every turn.

The human can disable at any time by saying **"cheapskate off"**. Re-enable with **"cheapskate on"** or **"cheapskate mode"**.

Every model response costs a premium request. **Subagent invocations do not cost additional premium requests** — verified by experiment. However, subagents still add latency and context overhead, so use them only when genuinely necessary.

---

## Model Tier Awareness

Not all models cost the same. Before doing anything expensive, know what tier you're on.

### Known Model Tiers (GitHub Copilot premium request multipliers)

> **This table is static** — it was accurate at time of writing but billing can change. On first load each session, ask the user if the rates still look right, or check: https://docs.github.com/en/copilot/managing-copilot/monitoring-usage-and-spending/about-premium-requests

| Multiplier | Models |
|------------|--------|
| **x1 (baseline)** | Claude Sonnet 4.5, Claude Sonnet 4.6, GPT-4o, GPT-4o-mini, Gemini 1.5 Pro |
| **x5–x10** | Claude Opus 4, o1, Gemini 2.0 Pro |
| **x15–x20** | o3, o3-mini (high), Claude Opus 3 |

> This table is static knowledge — verify against current Copilot pricing docs if rates change.

### Rules

- **x1 model**: no warning needed.
- **>x1 model**: warn the user at session start: _"You're on [model] (x[N] rate). In cheapskate mode, I'll minimize calls."_
- **Subagents cannot use a more expensive model tier than the parent agent** — verified by experiment. No MC/QQ gate needed for subagent model cost; they are capped at your current tier.

### Subagent Model Selection

Subagents cannot use a more expensive model than the parent session — the platform enforces this. You may still pass `model` explicitly for clarity, but omitting it is safe; the system will not silently upgrade to a pricier tier.

---

## Tool Cost Hierarchy

Rank tools from cheapest to most expensive and prefer cheaper options when they suffice:

| Tier | Tools | Cost |
|------|-------|------|
| **Free** | `read_file`, `list_dir`, `file_search`, `grep_search`, `get_errors`, `memory`, `run_in_terminal` | No AI inference |
| **Moderate** | `semantic_search` | Embedding model call |
| **No extra cost** | `runSubagent` | No additional premium request — but adds latency/overhead |

**Rules:**
- Use `grep_search` (exact/regex) before `semantic_search`. Only use `semantic_search` when you genuinely don't know what text to look for.
- Use `file_search` for locating files by name/pattern — never burn a `semantic_search` for that.
- Use `read_file` directly if you already know the path — no need to search first.
- Use `runSubagent` (Explore agent) only for deep multi-file exploration that would clutter the main conversation. Not for simple lookups.

---

## Parallelism — Do More Per Turn

Every additional turn = another premium request. Collapse multiple steps into single turns:

- **Parallel reads**: Call `read_file`, `list_dir`, `grep_search` in parallel when results don't depend on each other.
- **Parallel writes**: Use `multi_replace_string_in_file` instead of sequential `replace_string_in_file` calls.
- **Front-load context gathering**: Decide what you need to read, then read it all at once. Don't read one file, find you need another, read that — chain reads in parallel from the start.

---

## Context Discipline — Get It Right Once

- **No speculative searches**: Only search for things you've decided you need. Don't "explore to see what's there" unless the task demands it.
- **No revisiting**: If you already found the answer, don't search again to confirm. Trust your findings.
- **Stop when you have enough**: 2–3 search results that converge = sufficient context. More searching costs more without adding confidence.
- **Scope reads**: Use `startLine`/`endLine` on `read_file` to read only the relevant section. Don't read a 2000-line file to find a 10-line function.

---

## Turn Reduction Strategies

- **Batch file edits**: If you're making 5 edits to 3 files, plan them all and execute in one `multi_replace_string_in_file` call rather than 5 sequential calls.
- **Combine context gathering**: Don't make a tool call just to orient yourself — combine orientation with the actual read you need.
- **Skip redundant validation**: After a `replace_string_in_file`, trust the result unless there's a specific reason to re-read. Use `get_errors` once rather than reading the file back.
- **Use memory**: If you've computed or discovered something this session, write it to session memory rather than re-discovering it next time you need it.

---

## What NOT to Do (Anti-Patterns)

- **Cascade searching**: `grep_search` → "let me also check..." → `semantic_search` → "and one more..." — Stop after the first useful result.
- **Confirmation reads**: Reading a file you just wrote to confirm the write succeeded — trust the tool.
- **Speculative subagents**: Launching Explore for a question you could answer with a single `grep_search`.
- **Small sequential reads**: `read_file` lines 1–50, then 51–100, then 101–150 — read the full range in one call.
- **Redundant re-reads**: Reading the same file twice in one task because you forgot you already read it — check session memory first.

---

## User Input — Eliminate Typing Turns (MC/QQ)

> **MC/QQ** = Multiple Choice / Quick Question box — the `vscode_askQuestions` tool, which renders clickable buttons in the chat UI at zero cost.

Every time the human has to type a free-form answer to a question you asked, that's a wasted turn — they had to stop, think, and type instead of clicking.

**Rule: NEVER ask a yes/no or multiple-choice question in plain text. Always use `vscode_askQuestions` with options.**

```
vscode_askQuestions([
  {
    header: "Proceed",
    question: "Ready to post this comment to ADO?",
    options: [
      { label: "Yes, post it", recommended: true },
      { label: "No, cancel" }
    ]
  }
])
```

Benefits:
- The human clicks instead of types — faster for them, no parsing overhead for you.
- Avoids ambiguous answers ("yeah sure" vs "yes" vs "go ahead") that could cost a follow-up turn.
- Forces you to enumerate the options upfront, which clarifies your own thinking.

Apply to: approval gates (Gerrit push, ADO comment), "which approach do you prefer?", "should I continue?", and any other decision point.

**Anti-pattern**: "Should I proceed? Let me know!" — this forces the human to type. Use `vscode_askQuestions` instead.

**`vscode_askQuestions` is free** — it's a VS Code UI tool that renders a button in the current response. No model inference, no extra premium request. It costs nothing.

### End-of-Response Rule

**At the end of every response, always try to close with a `vscode_askQuestions` call.**

However, adapt frequency to the session flow:

| Flow | MC/QQ Frequency |
|------|----------------|
| **Collaborative** (user is engaged turn-by-turn, no broad pre-approval) | Ask freely — mid-response for choices, end-of-response for next step |
| **Autonomous** (user has pre-approved a scope, "just do it" mode) | Gate only at the end of the full task or at hard, irreversible decision points — not for every small step |

### Detecting the Flow

You can't ask the user directly every session — that defeats the purpose. Read the signals:

**Autonomous signals** (suppress mid-response MC/QQ):
- Explicit broad grants: "just do it", "go ahead", "handle it all", "you have my permission", "do everything"
- Multi-step task handed off in one message: "implement X, Y, and Z"
- Running a Pulse or Dream cycle
- User has said "stop asking me, just do it" at any point this session

**Collaborative signals** (use MC/QQ freely):
- Short, specific single requests
- User is reviewing and commenting on each step
- User is asking clarifying questions back
- No broad pre-approval has been given

When in doubt, default to **collaborative** — it's safer and cheaper than burning turns on wrong assumptions.

If you're uncertain mid-task, use one MC/QQ to resolve it: _"Do you want me to continue autonomously, or check in after each step?"_

In autonomous flow, intermediate MC/QQ boxes are disruptive. Batch all small decisions and ask once at a natural break point, or only when something falls outside the pre-approved scope.

Ask: is the result good, does anything need tweaking, or what should come next? This eliminates a free-text follow-up turn and keeps the human in a click-to-continue flow.

```
vscode_askQuestions([
  {
    header: "Next",
    question: "Does this look good, or do you want to adjust something?",
    options: [
      { label: "Looks good", recommended: true },
      { label: "Tweak something" },
      { label: "Move on to the next task" }
    ]
  }
])
```

Skip it only when: the response is a pure factual answer with no logical follow-up, or a question was already asked inline during the response.

---

## When to Suggest Cheapskate Mode to the Human

If you observe the task sprawling into many turns with no clear end:
- Pause, state what you know, and ask the human to confirm the exact goal before continuing.
- Propose a plan with clear steps so the human can redirect before you spend 10 more turns on the wrong approach.

---

## Summary Checklist (Apply Before Each Tool Call)

1. Is there a cheaper tool that would get the same result?
2. Can I batch this with other reads/writes in the same tool call?
3. Do I already have this information from earlier in the conversation?
4. Am I reading more than I need to?
5. Am I about to launch a subagent for something I could do inline?
