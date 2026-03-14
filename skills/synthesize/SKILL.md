---
name: synthesize
description: "TRIGGER when the user wants to distill, synthesize, or consolidate notes into a structured knowledge note. Use for: turning rough thinking notes into polished concept notes, processing brainstorm outputs (ChatGPT, cofounder, VC conversations) into structured takeaways, converting course material into revision-ready notes. Signal phrases: 'synthesize these notes', 'distill this into a knowledge note', 'turn this brainstorm into something structured', 'create a concept note from', 'process these ideas'. Also triggers for /synthesize. NOT for project syncing (use /project-sync), meeting transcript processing (use /meeting), or decision analysis (use /decision)."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, synthesis, knowledge, concept, brainstorm]
---

Take the notes or content I specify and distill them into a structured knowledge note.

## Arguments

This command accepts file paths as arguments — one or more paths to files in the vault to synthesize.

Example: `/synthesize 03_Thinking/some-idea.md 01_Execution/2026-02-18.md`

Read all specified files before proceeding. If no arguments are provided, ask the user what content to synthesize.

## Step 1 — Identify the kind

Choose the right template based on what was provided:

- **concept** — a paper, article, idea, or external resource being processed into settled understanding
- **course** — revision material for an academic course (e.g. NotebookLM output, cheat sheet, past paper patterns)
- **brainstorm** — output from an AI session (ChatGPT, Gemini) or a conversation (co-founder, VC, collaborator)

If unclear, default to **concept**.

## Step 2 — Write the note using the matching format

### concept
```
---
type: knowledge
kind: concept
source: [title or URL if applicable]
date: [today]
tags: []
---

# [Concept / Topic]

## What it is
Core idea in 2–3 sentences.

## Why it matters
Relevance to the user's work or thinking.

## Key points
-

## Open questions
-
```

### course
```
---
type: knowledge
kind: course
course: [course name]
topic: [specific topic]
date: [today]
exam: [exam date if known]
---

# [Course] — [Topic]

## Core concepts
What you need to understand, not just memorize.

## Exam patterns
What keeps showing up in past papers.

## Gotchas
What's easy to get wrong under pressure.

## Quick reference
Formulas, definitions, key facts.
```

### brainstorm
```
---
type: knowledge
kind: brainstorm
project: [related project if applicable]
date: [today]
with: [chatgpt | gemini | co-founder | vc | solo]
---

# [Topic] — Brainstorm

## Question being explored

## Best ideas surfaced
-

## Dead ends
What didn't work or felt wrong.

## Threads to pull
Open directions worth pursuing further.
```

## Language

Write in English by default. If the source material is predominantly in Chinese and the user has not specified a preference, ask which language to use. Frontmatter field names and YAML keys stay in English regardless.

## Step 3 — Save location

Save the note to `04_Knowledge/[subfolder]/[descriptive filename].md`. **Always use a subfolder — never save directly to `04_Knowledge/` root.**

Choose the subfolder by matching the content to an existing folder or creating a new one:
- **Project match (highest priority)** — if the frontmatter `project:` field is set, **always** use the project folder (e.g. `04_Knowledge/ProjectAlpha/`). This ensures `/project-sync` can find the note.
- **Domain/topic match** — if no project applies, use or create a thematic folder (e.g. `04_Knowledge/Hardware/`, `04_Knowledge/AI/`, `04_Knowledge/Finance/`)
- **If in doubt**, infer from the source file's location, filename, or content tags

Create the subfolder if it doesn't exist.

## Step 4 — Report back

- Knowledge note saved to: `[path]`
- One-sentence summary of the synthesis.

---
**Next steps:**
1. Run `/project-sync <project>` to include this note in the project's Knowledge Base
