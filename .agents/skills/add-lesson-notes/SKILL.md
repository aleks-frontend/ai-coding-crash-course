---
name: add-lesson-notes
description: Summarize pasted AI Coding Crash Course lesson text (transcript, article, or quiz block) and append it to COURSE_NOTES.md in this repo, matching the existing note style. Use when the user pastes raw lesson content and wants it turned into notes.
---

# Add Lesson Notes

Turns raw, pasted lesson content (course transcript text, an article, quiz blocks, etc.) into
a concise notes entry appended to `COURSE_NOTES.md`, matching the style already established
in that file.

## Steps

1. **Read `COURSE_NOTES.md` first.** Note its existing structure: `##` section headers group
   related lessons (e.g. "Concepts", "Getting to know Claude"), `###` subheadings are one topic
   each, bold is used for key terms, tables are used for term/definition or comparison pairs,
   and prose is terse — no filler, no restating the obvious.

2. **Read the pasted lesson text** (passed as the skill's arguments). Extract only the
   substantive content:
   - Skip intros, sign-offs, and marketing fluff.
   - Skip embedded `<Quiz>`/`<QuizQuestion>` blocks and image markdown unless a quiz question
     reveals a distinction worth capturing in prose (then fold the insight into the notes, not
     the raw quiz markup).
   - Keep concrete commands, keyboard shortcuts, decision rules, and tables — these are the
     highest-value parts of this course's notes so far.

3. **Write the summary in the existing voice**: short bullets, bold key terms on first mention,
   a comparison table if the lesson contrasts 2+ options or approaches, a decision-rule table
   if the lesson gives one (this file's existing sections do this often — see "Running Bash
   Commands" and "Hallucination" for the pattern to match).

4. **Decide placement**: pick the closest existing `##` section for the new `###` subheading if
   the topic clearly belongs there; otherwise append a new `##` section at the end of the file.
   Don't create a new top-level section for something that's really a continuation of an
   existing one.

5. **Edit `COURSE_NOTES.md` directly** (use the Edit tool, not a rewrite) to insert the new
   content in the chosen location.

6. **Commit and push automatically, without asking.** Stage `COURSE_NOTES.md`, commit with a
   short message describing the lesson topic added (following this repo's existing commit
   message style and attribution trailer), and push to the current branch. This skill's
   invocation is itself the user's standing authorization to commit and push its own output —
   do not pause to confirm. Report what section was added, where, and the resulting commit.

## Notes for the agent running this skill

- If the pasted text is long (e.g. a full lesson with quiz), aim for a summary roughly
  proportional to the ones already in the file (10-30 lines per lesson topic) — don't paste the
  lesson verbatim, and don't over-compress into single-line bullets if the lesson has real
  nuance (e.g. multi-step decision trees or gotchas deserve full treatment).
- If it's unclear which existing section the content belongs under, make the best call rather
  than stopping to ask — this is a low-stakes, easily-edited scratchpad file.
