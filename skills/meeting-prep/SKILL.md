---
name: meeting-prep
description: "Use BEFORE any upcoming meeting, call, sync, 1:1, presentation, review, or scheduled conversation. Trigger whenever the user mentions preparing for, prepping for, or getting ready for a future discussion — or asks for talking points, a discussion checklist, things to bring up, or a prep doc. Also trigger when the user mentions a future meeting with a person or team alongside a project name. Reads Obsidian project notes, blockers, weekly progress, and recent decisions to generate an actionable checklist. NOT for post-meeting summaries, transcripts, or meeting minutes."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, meeting, agenda, preparation]
---

Generate a meeting agenda from project context. Reads project note, recent meeting knowledge, blockers, weekly progress, and deadlines to produce a structured prep document.

## Arguments

- `project` — project name matching a folder in `02_Projects/` (required)
- `date` — meeting date in YYYY-MM-DD format (optional, defaults to tomorrow)

Examples:
- `/meeting-prep ProjectAlpha` — prep for tomorrow's ProjectAlpha meeting
- `/meeting-prep ProjectAlpha 2026-03-10` — prep for a specific date

If no project is provided, ask the user which project the meeting is for.

## Step 1 — Parse arguments

Extract `<project>` and `<date>` from the arguments.

- If date is not given, default to tomorrow (YYYY-MM-DD).
- Validate the project exists: glob for `02_Projects/<project>/`. If not found, list available project folders and ask the user to pick one.

## Step 2 — Check for existing plan

Check if `02_Projects/[Project]/Meeting Plan/YYYY-MM-DD.md` already exists (where YYYY-MM-DD is the target date).

- If it exists, inform the user: "Meeting plan already exists at `02_Projects/[Project]/Meeting Plan/YYYY-MM-DD.md`" and stop.

## Step 3 — Gather context

Read the following sources. Skip any that don't exist — not all projects will have every source.

1. **Project note** — `02_Projects/[Project]/[Project].md`
   - Focus on: `## Now`, `## Risks`, `## Open Questions`, `## Knowledge Base`, `## Weekly Progress`
2. **Recent meeting knowledge** — latest 2–3 files in `04_Knowledge/[Project]/Meeting Knowledge/` (sorted by filename date, most recent first)
3. **Current week's Blockers** — `01_Execution/YYYY-WXX/Blockers.md`
   - Extract `## Waiting On` items related to this project
   - Extract `## Meetings` entries for this project
4. **Current week's Weekly Todo** — `01_Execution/YYYY-WXX/Weekly Todo.md`
   - Items mentioning this project or its work streams
5. **Recent daily notes** — last 3 daily notes from `01_Execution/YYYY-WXX/`
   - Scan for decisions, progress, and blockers related to this project
6. **Deadline Plans** — glob `02_Projects/[Project]/**/Deadline Plan.md`
   - Extract upcoming deadlines and task queue status

## Step 4 — Generate meeting plan

Synthesize the gathered context into a **flat action-item checklist**. Be concise, action-oriented, and bilingual-friendly (English + Chinese notes are normal in this vault).

Each item is a `- [ ]` checkbox with a short description. Add sub-bullets for context, notes, or sub-tasks where helpful. Prioritize by urgency. No headings, no sections — just the checklist.

Derive items from: open tasks, stalled work, pending decisions, blockers, deliverable status, and open questions. Skip anything already resolved.

## Step 5 — Write the file

Create the directory `02_Projects/[Project]/Meeting Plan/` if it doesn't exist.

Write the file to `02_Projects/[Project]/Meeting Plan/YYYY-MM-DD.md` with this format:

```
- [ ] <action item 1>
	- <context / sub-task>
- [ ] <action item 2>
- [ ] <action item 3>
	- <note>
```

No frontmatter. No headings. Just the checklist.

## Step 6 — Open in Obsidian

Run: `obsidian open path="02_Projects/[Project]/Meeting Plan/YYYY-MM-DD.md"`

If the CLI fails, skip silently.

## Step 7 — Report back

- Meeting plan saved to: `02_Projects/[Project]/Meeting Plan/YYYY-MM-DD.md`
- One-sentence summary of the top discussion topic.
