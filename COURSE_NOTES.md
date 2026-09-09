# AI Coding Crash Course — My Notes

**Course:** [AI Coding Crash Course](https://www.aihero.dev/workshops/ai-coding-crash-course) by Matt Pocock (AI Hero)
**Started:** 2026-08-25
**Repo:** `ai-coding-crash-course` — mini-Udemy-style course platform (React Router v8, TypeScript, SQLite/Drizzle, Tailwind + shadcn/ui, Vitest) used as the hands-on exercise codebase.

Personal scratchpad for notes, gotchas, and things worth remembering as I go through the course.

---

## Some concepts we will cover

- smart zone / dumb zone - learn to recognize when agent is going dumb and how to keep it in smart zone
- managing context
- compaction vs. handoff - 3 ways to continue past a full context
- grilling
- codebase exploration
- AGENTS.md
- skills
- progressive disclosure
- crystal clear requirements
- decompressing complex work into session-sized chunks
- handoffs - leaving breadcrumbs for an agent so we can both pick up on where we left later
- subagents - how to delegate exploration and other tasks
- ...

---

## Notes

<!-- Add lesson notes, insights, and questions below as you go. -->
Writing `/model` gives you a list of available models:
- up/down arrows change model
- left/right can change effort
- control+Shift+G (jump to git message)

## Before we start

### Project commands

- npm run reset (reset to a certain lesson)
- npm run cherry-pick (resets to a certain lesson but keeps your custom changes)
- npm run pull (pull latest course updates)

### Database migrations

![alt text](image.png)

Migrate === Updates the database to how the source code says it should look

![alt text](image-1.png)

npm run db:migrate

### Reseting to a clean slate

Before the course we will reset all our Claude settings and back it up.

I didn't do a full cleaning of my existing projects (think of doing that after the end of the course).

[IMPORTANT] Restore command: "restore my Claude Code config 
  from ~/agent-config-backup-2026-08-26"

### Installing the Request logger

In one terminal just run this: npm run request-logger

And in another one just paste what logger suggested.
Example: `ANTHROPIC_BASE_URL=http://localhost:8787 ENABLE_TOOL_SEARCH=true claude`

This will be generating log files in `request-logger/logs` - we will be checking these in more detail later in this course.

---

## Concepts

### Models, Harnesses, Agents, Environments

![alt text](image-2.png)

**An Agent is a model, harnessed in an environment.**

Examples: 

- *Opus 4.8, on Claude Code, on the filesystem*
- *GPT 5.5, on ChatGPT, in the ChatGPT environment*

### Turns and Model Provider Requests

Used the request logger (`npm run request-logger`) to see the actual traffic between the harness (Claude Code, local) and the model provider (Anthropic, in the cloud). Logs land in `request-logger/logs`, one file per request.

- **Model provider request** = one round trip: request out, response back. Handled by a **model provider** (Anthropic, Ollama, etc).
- Each log file shows: meta/headers, the full **system prompt**, tool definitions (JSON schema), the conversation so far, and the response.
- A simple "hello" already sends a huge system prompt + tool definitions for a one-line reply.
- **The full conversation history is resent on every request** — previous messages, previous responses, all accumulated context. Nothing is summarized under the hood by the model; this is why logs (and cost) grow as a conversation goes on.
- Asking for something simple (e.g. "tell me my hair looks nice") can trigger *multiple* hidden requests, not just one — e.g. background suggestion requests (`SuggestionMode`) that generate what you might type next. Every AI-generated bit of UI = a model provider request happening somewhere.
- **Tool call** = the model's instruction to use a tool (e.g. `write` with `file_path` + `content` params) — this is not execution, just a request.
- **Tool result** = the output of the harness actually running that tool (e.g. writing the file to disk). The model never touches the filesystem directly; the harness executes tool calls and reports results back to the model in the next request.
- **Turn** = the entire cycle from a user message to the agent's final answer, including every model provider request, tool call, and tool result along the way. A single turn can contain hundreds of model provider requests and last hours.

| Element | Definition |
|---|---|
| Model Provider Request | One round trip to the model (request + response) |
| Tool Call | The model's instruction to use a tool (not the execution) |
| Tool Result | The output of executing that tool |
| Turn | The entire cycle from user message to agent response, including all tool calls and results |

### Sessions and the Context

**Context** = all the available information the agent has access to at request time (instructions, parameters, scope, tool calls, conversation so far). It gets turned into tokens, and the model does next-token prediction on top of it to produce its response.

- **Context window** = the max tokens a model can receive in a single request. It's an arbitrary number picked by the model's developers based on what their infra can handle (e.g. GPT 5.6 Sol: 1.1M tokens, Gemini 3 Pro: 1M tokens, GPT 5.2 Chat Latest: 128k tokens).
- Go over the context window and **the request just fails** — no output tokens at all. Nothing gets silently trimmed/dropped for you.
- You're billed for every token sent, every request, so keep context relevant — irrelevant text costs money AND can distract/confuse the model, same as it would a human.
- **The context window never starts at zero when using a harness.** The harness (agentic framework/wrapper) injects a **system prompt** right from the start: what tools are available, high-level instructions, what role the agent should inhabit. Different harnesses send very different amounts of this text.
- **Session** = a continuous conversation with an agent where context builds up over multiple turns (turn 1: 5 model provider requests, turn 2: 3 requests, turn 3: 4 requests... all part of the same session, each new request carrying the full history so far).
- Having the word "session" matters because it's the unit you act on: you can **clear** it (start fresh), **compact** it (shrink but keep going), or set it aside and come back later.

| Term | Definition |
|---|---|
| Context | All the available information the agent has access to, including instructions, parameters, scope, and tool calls |
| Context Window | The maximum amount of context (in tokens) that a model can receive in a single request |
| System Prompt | The initial instructions given to the agent by the harness, defining its role and behavior |
| Session | A continuous conversation with an agent, where context builds up over multiple turns |

### Smart Zone / Dumb Zone

A typical agent turn uses ~18k tokens, but models advertise context windows up to 1M — so why not just use all of it? Because the model doesn't reason over that text for free: every token added has to be related to every other token already in context (an **attention relationship**), and that count scales quadratically, not linearly.

- 2 tokens → 1 relationship, 3 tokens → 3, 4 → 6, 5 → 10.
- At scale: 1,000 tokens ≈ 1 million relationships, 10,000 tokens ≈ 100 million, 100,000 tokens ≈ 10 billion.
- The more relationships the model has to track, the worse its **attention** performs — this is **attention degradation**. It's the underlying mechanism and doesn't go away as models improve; only the thresholds shift.

This produces two zones within a single context window:

- **Smart zone** (early in the context window) — agent handles planning, complex builds, strategic decisions well.
- **Dumb zone** (context filled up) — agent still "works" but gets sloppy even on simple stuff (basic file writes, closing issues, writing specs). Also costs more per request since you're sending huge amounts of tokens each time.

- Current (debated, moves with model quality) consensus: dumb zone starts around **150k tokens** — up from ~100–120k in earlier course versions. Expect this number to keep climbing as models improve.
- The 1M-token advertised window ≠ usable smart-zone size. It exists because (a) it's a good headline, and (b) some use cases (pure retrieval over long text) don't need peak reasoning. Coding work does need the smart zone.
- **It's a slope, not a cliff** — quality degrades gradually as context fills, not suddenly at 150k. Treat ~150k tokens in a session as the signal to start planning a bail-out: hand off the work, compact, or otherwise get back into a fresh smart zone rather than grinding on inside the dumb zone.

### Statelessness

The **model** itself is completely stateless — it processes a single request based only on the context it's handed, and retains nothing between requests. All the "memory" of a conversation actually lives one level up.

| Concept | Stateful? | Scope | Responsibility |
|---|---|---|---|
| Model | Stateless | N/A | Processes a single request based on provided context |
| Harness | Stateful | Within one session | Remembers all messages and session history |
| Environment | Stateful | Indefinite | Persists files, changes, and data on disk |

- The **harness** carries the session's message history forward across turns, resending it all each request — but only for the life of that session. **Clear** the session and the harness forgets everything.
- The **environment** (filesystem, etc.) is the only thing that's *always* stateful — save a file, then clear the session, and the file is still there.
- People who want the agent to "remember stuff about them" over time are really asking for state in a stateless system — that's what memory systems bolt on. But the simpler, usually-sufficient answer is: **save it in the environment** (i.e. the codebase) rather than reaching for a separate memory system. Quoting Mario Zechner (creator of Pi): *"my codebase is my memory system."*
- Default to statelessness where you can — it's simpler and tends to work better than bolting memory onto the agent itself.

### Hallucination

**Hallucination** = confidently wrong model output. It's the main thing to guard against — it can produce broken code, steer the product in the wrong direction, or recommend insecure/outdated APIs. Symptom of the dumb zone, but can occasionally show up in the smart zone too.

Two flavors:

| Type | What it is | Cause | Fix |
|---|---|---|---|
| **Factuality** | Invented/wrong facts (nonexistent function, wrong API signature, fake citation) | Lack of **parametric knowledge** — info not in context, so the model dredges from training | Load **contextual knowledge** — never trust an unsourced LLM, give it a source |
| **Faithfulness** | Model ignores, drifts from, or misuses info that *is* in its context | **Attention degradation** in the dumb zone — too much context to find what's relevant | Clear the context window, get back to the smart zone |

- **Parametric knowledge** = knowledge baked into the model's weights during training. It's stored as "fuzzy vibes," not a lookup table — compressing terabytes of data into a few billion numbers loses detail, so retrieval from memory is unreliable even for pre-cutoff facts.
- **Knowledge cutoff** = the point where a model's training data ends. Can't be patched — updating a model's knowledge requires retraining from scratch. Anything after the cutoff, or anything fuzzy from before it, is a factuality risk.
- Example: asked without web search, Claude Opus gave outdated X API pricing tiers from training data — a factuality hallucination. Web search corrected it by supplying contextual knowledge.
- Contextual knowledge reduces factuality hallucinations but isn't a full cure, since faithfulness hallucinations can still happen even with the right info sitting right in context.
- **Decision tree** when you spot a hallucination: was the info in the context window?
  - **No** → factuality problem → load sourced info into context.
  - **Yes** → faithfulness problem → clear context to reduce attention degradation.

### Subagents

A common harness technique to fight smart-zone/dumb-zone constraints: **delegate work to another agent** instead of doing everything in the main agent's context.

- A session's context breaks into phases (e.g. system prompt, exploration, implementation). Making any phase cheaper (fewer tokens) means less money spent and more room left in the smart zone before attention degradation kicks in — but cutting corners on exploration risks worse info feeding the implementation phase.
- **Subagent** = an agent spawned by the main agent (the **orchestrator**) to do a task — e.g. deep codebase exploration — and report back just a summary. Like a senior dev asking a junior to research something and report findings: the subagent burns its own tokens digging in, but only the summary lands in the orchestrator's context.
- Subagents can be spawned **in parallel** — multiple at once, each researching something different, all reporting back to the orchestrator.
- Each subagent can be configured independently: different **system prompt**, different **model**, different **effort** level. Lets you send a cheap/fast subagent at mechanical search and a stronger one at a hard question.
- **Recursive subagents**: a subagent can spawn its own subagents (harness-dependent — some allow only one level deep, some allow full recursion).
- Net effect: subagent tokens are still spent, just isolated from the orchestrator's window — the benefit is context efficiency for the main agent, not free computation.

---

## Getting to know Claude

### Managing Your Claude Code Session

Running Claude Code from VS Code's integrated terminal for this course.

- Just run `claude` in the terminal to open the chat UI — type a message like any chat app (e.g. "hello, how are you?").
- `/terminal-setup` — sets up key bindings, notably **Shift+Enter** for multi-line prompts. Run once; may need manual setup on WSL.
- `/usage` — shows remaining allowance: current session usage, weekly limit, current usage against it. Once you hit the limit you can't use the agent until it resets. Press Escape to exit back to chat.
- `/context` — visualizes what's filling the context window: system prompt tokens, skills tokens, message tokens, and total context window size. Useful for debugging what's eating your budget.
- `/clear` — empties the conversation history, giving a fresh context window. Confirms the model is **stateless**: once cleared, the agent has no memory of what came before (e.g. ~9,000 tokens with history → ~6,600 tokens after clear). Ctrl+C twice also opens a brand-new session with no memory.
- **Escape** — interrupts the agent mid-run, canceling everything it was doing, and leaves an `interrupted` marker so you can redirect it (conversation stays intact, unlike `/clear`). Say "carry on" to resume if you interrupt by accident.

### Showing Context Usage in the Status Line

Claude Code doesn't surface context-window usage by default — `/context` is a manual, one-off check. For an always-visible number, use [`ccstatusline`](https://www.npmjs.com/package/ccstatusline), a community tool that reads Claude Code's session data and renders a custom status line.

Setup:

1. `mkdir -p ~/.config/ccstatusline`
2. Create `~/.config/ccstatusline/settings.json` configuring the widgets you want (e.g. a bold yellow `context-length` token count, a dimmed `context-percentage` in parentheses right after it — `"merge": "no-padding"` glues adjacent widgets together, `"rawValue": true` strips labels down to plain numbers).
3. In `~/.claude/settings.json`, add (preserving any existing keys, e.g. the `permissions.deny` / `disable*` bloat-cutting settings from earlier):

```json
{
  "statusLine": {
    "type": "command",
    "command": "npx ccstatusline@latest"
  }
}
```

4. Fully quit and reopen Claude Code (not just `/clear`).

Result: status line shows something like `186.2k (17.3%)`, live-updating as context fills — a lightweight companion to the [smart zone / dumb zone](#smart-zone--dumb-zone) awareness habit, so you don't have to remember to run `/context` manually.

### Prompting in the Terminal

- **`@` file picker** — press `@` to fuzzy-search and pull files into your prompt (arrow keys to browse, tab/return to select, repeat for multiple files). Files are read straight into the context window on the first request — no extra tool call needed. Great for handing the agent a spec or config file to follow directly.
- **`Ctrl+S` to stash a prompt** — holds a half-written prompt aside so you can send something else first (e.g. a quick correction), then rehydrates the stashed draft back into the text box when you're ready. Useful for giving feedback mid-thought without losing your original message.
- **Pasting images** — copy an image (including screenshots) and paste it directly into the text box; a "pasting" indicator confirms it's attached. The image becomes part of the prompt, so the agent can analyze screenshots/mockups without a separate upload step.

### Running Bash Commands

Three ways to run bash commands with the agent, depending on whether you want it to see the output:

- **Bash mode (`!` prefix)** — runs a command and puts its output straight into the agent's context. Use when the agent needs to act on the result (e.g. `! npm run typecheck` so it can fix the reported errors).
- **Backgrounding (Ctrl+B)** — for long-running processes that never exit on their own (dev servers). Started in bash mode, then backgrounded with Ctrl+B; output streams to a local file and a background task entry appears under the status line (view, or stop with X). Lets the agent debug against live server logs — try something in the UI or send a curl request, then check what the server logged — without blocking the session.
- **Suspending (Ctrl+Z, then `fg`)** — suspends the whole agent so you can run a command it never sees, preserving its state. `fg` brings it back exactly as it was. Useful for commands whose output you don't want in context at all.

| Goal | Approach | Shortcut |
|---|---|---|
| Agent needs to see the output | Bash mode | `!` prefix |
| Long-running process (dev servers) | Background it | Ctrl+B after `!` command |
| Command hidden from the agent | Suspend the agent | Ctrl+Z, then `fg` to return |

### Permissions

Claude Code is strict by default about what the agent can do without asking — a safeguard against an agent with unlimited power doing something dangerous by accident (e.g. deleting the whole filesystem).

**The approval flow** — a safe command (`echo hello`) just runs. A riskier one (`pnpm typecheck`) triggers a **permission request** showing the exact command, why it wants to run it, and three choices:

1. **Allow once** — permitted this one time only.
2. **Allow always** — permitted from now on for this project (adds a rule like `Bash(pnpm typecheck)` to settings).
3. **Reject and suggest** — deny it and propose an alternative command instead (e.g. reject `pnpm typecheck`, suggest `npx tsc`; approving that with "allow always" adds `Bash(npx tsc *)`).

**Where permissions live** — `.claude/settings.local.json`, editable by hand ahead of time:

```json
{
  "permissions": {
    "allow": ["Bash(pnpm typecheck)"]
  }
}
```

- Wildcards widen a rule: `"Bash(pnpm *)"` allows all `pnpm` commands.
- A `deny` array blocks a command outright regardless of what the agent (or a classifier) thinks — e.g. `"Bash(git push *)"` to always block pushes. This is the only reliable way to hard-block something; a prompt instruction is not a permission rule.

**Not just Bash** — web search and web fetches need permission too (e.g. approving a fetch adds `WebFetch(domain:reactrouter.com)`), so the agent can back up local reasoning with real docs instead of guessing.

**Sharing with a team** — `settings.local.json` is gitignored, so approvals stay local to you. Rename it to `settings.json` and check it in, and anyone who clones the repo and runs the agent inherits the same allowed/denied commands immediately — no manual setup per teammate.

**Auto mode vs. manual** — the mode selector (bottom-left, cycle with Shift+Tab) offers Manual, Edits, Plan, and **Auto** mode:

| Mode | Behavior |
|---|---|
| Manual | Every non-trivial command needs explicit approval |
| Auto | An LLM classifier (likely Claude Haiku) judges each command's safety from the conversation and runs it without asking if deemed safe |

- Auto mode costs a little time/tokens per command (the classifier call itself), but removes most manual interruptions.
- The classifier is imperfect in both directions — it can allow things you'd rather it didn't (e.g. a database migration) and block things that are actually fine (e.g. creating a GitHub issue) — but reliably blocks the always-bad stuff (`rm -rf` on your filesystem).
- **`settings.json` is checked first, before the classifier runs** — so explicit allow/deny rules still take priority and save the classifier round-trip for common commands, even in auto mode.
- Set auto mode as the default via `/config` → **default permission mode**.

---

## Fundamentals

### Starting Context: Resetting Your Config

Before doing any real work with an agent, check what's already sitting in the context window by default — default configs (MCP servers from claude.ai like Figma, Gmail, Google Calendar, Google Drive, Slack, Todoist, Zapier; built-in skills) can add a surprising amount of bloat before you've typed a single real message.

- **Reset to defaults**: rename `.claude/settings.json` → `settings-backup.json`, and `.claude/skills` → `skills-backup`. This strips custom config back to the harness's defaults so you can see the true baseline.
- **`/context`** — run it before sending any real messages to see the baseline breakdown (system prompt, system tools, MCP tools, skills, messages, total). Example from the lesson: default config ≈ **23k tokens** baseline; after restoring a trimmed custom config (fewer MCP servers, fewer skills) ≈ **6.6k tokens** — a ~16k token difference before any real work starts.
- **Why this matters isn't cost** — it's [smart zone](#smart-zone--dumb-zone) space. Every token spent on config bloat is a token not available for the agent to actually reason in. Trimming config doesn't change the size of the context window itself; it changes how much of that window is free for reasoning versus already consumed by setup.
- Practical habit: periodically audit your own `.claude` config (and other people's, when reviewing) for MCP servers and skills that aren't actually needed for the work at hand.

### Killing Bloat: Cutting the System Prompt Payload

Continuation of the baseline-audit habit above, but going deeper: using the **request logger** to see the literal payload sent on every request, then cutting it down via `~/.claude/settings.json`.

**Workflow for each settings change**: edit `~/.claude/settings.json` → quit the agent → relaunch → send `Hello!` → run `/context` → check the logs for what changed.

**Inspecting the raw payload** (via `request-logger/logs/*.md`):
- `<system-prompt>` — environment section (OS/shell/cwd), context-management instructions, recent git commits.
- `<tools>` — every tool's full JSON schema ships on every request. Expect 70+ tools, many obscure (`CronCreate`, `DesignSync`, `Workflow`) and verbose (redundant commit-message/PR-template text).
- `mcp__` — MCP connector tools (Figma, Gmail, Slack, Google Drive, etc.) ship whether or not you use them.
- "following skills are available" — the skills catalogue, 15-20+ entries with descriptions, built-in + project-specific mixed together.

**Key principle**: denying a tool via `permissions.deny` doesn't just block it at runtime — it **removes the tool's definition from the system prompt entirely**, saving tokens on every single request, not just a one-time cost.

**Settings.json bloat-cutting sequence** (results from the lesson, starting at a fresh baseline of **68.3k tokens**):

| Setting | Effect | Running total |
|---|---|---|
| `"disableClaudeAiConnectors": true` | Drops all claude.ai MCP connectors | 47k |
| `"disableWorkflows": true` | Drops dynamic multi-subagent workflow config | 39k |
| `"disableBundledSkills": true` | Drops built-in skills (deep research, data viz, artifact design, etc.) | 37.1k |
| `"disableArtifact": true` | Drops the Artifacts feature | 33.1k |
| `permissions.deny` list (see below) | Removes unused tool definitions from the prompt | 21.6k |
| Add `AskUserQuestion` to the deny list | Removes its ~130-line tool definition | ~19.9k |

Tools worth denying if unused: `NotebookEdit`, `DesignSync`, `CronCreate`/`CronDelete`/`CronList`, `EnterPlanMode`/`ExitPlanMode`, `PushNotification`, `RemoteTrigger`, `ReportFindings`, `ScheduleWakeup`. `AskUserQuestion` is a judgment call — some like its UI, others find it intrusive; denying it is a straight token cut, not a correctness issue.

Full example config:

```json
{
  "permissions": {
    "deny": [
      "NotebookEdit", "DesignSync", "CronCreate", "CronDelete", "CronList",
      "EnterPlanMode", "ExitPlanMode", "PushNotification", "RemoteTrigger",
      "ReportFindings", "ScheduleWakeup", "AskUserQuestion"
    ]
  },
  "disableClaudeAiConnectors": true,
  "disableWorkflows": true,
  "disableBundledSkills": true,
  "disableArtifact": true
}
```

- The specific list is agent-specific (this is all Claude Code) — the transferable habit is: **audit your harness's payload, deny/disable whatever you don't actually use**, since it's pure cost (tokens + potential distraction) for zero benefit.
- None of this shrinks the context *window* — it shrinks how much of the window is pre-consumed by setup before you've typed a word, leaving more room in the [smart zone](#smart-zone--dumb-zone).

