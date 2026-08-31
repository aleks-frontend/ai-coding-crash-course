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


