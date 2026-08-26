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

## Project commands

- npm run reset (reset to a certain lesson)
- npm run cherry-pick (resets to a certain lesson but keeps your custom changes)
- npm run pull (pull latest course updates)

## Database migrations

![alt text](image.png)

Migrate === Updates the database to how the source code says it should look

![alt text](image-1.png)

npm run db:migrate

## Reseting to a clean slate

Before the course we will reset all our Claude settings and back it up.

I didn't do a full cleaning of my existing projects (think of doing that after the end of the course).

[IMPORTANT] Restore command: "restore my Claude Code config 
  from ~/agent-config-backup-2026-08-26"
