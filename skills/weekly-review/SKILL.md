---
name: weekly-review
description: "Invoke IMMEDIATELY when the user asks to review their week, generate a weekly review or weekly synthesis, check weekly progress, score intentions, or detect stalled/horizon items. Responds to /weekly-review. Triggers on: 'review the week', 'weekly review', 'weekly synthesis', 'how did this week go', 'generate weekly review for W##', 'do my weekly review', 'score my intentions', 'flag horizon items', 'what stalled this week'. Produces an Obsidian weekly review note with AI synthesis covering progress, stalled items, intention scorecards, horizon-item detection, and next-week focus; leaves reflection blank for user. Not for weekly planning/init, project-specific status checks, meeting transcript processing, or standalone next-week planning."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, weekly-review, synthesis, reflection]
---

Produce a structured weekly review note for the week just completed.

## Arguments

Optional: `last` (review last week) or `YYYY-WXX` (specific week). Default: current ISO week (or last week if run on Monday).

## Steps

1. **Determine the week** — Accept an optional argument: `last` (always reviews last week) or `YYYY-WXX` (specific week). If no argument, default to the current ISO week (or the most recently completed week if run on a Monday). Week folder format: `YYYY-[W]WW` (e.g. `2026-W08`).

2. **Read daily notes** — Read all daily note files in `01_Execution/YYYY-[W]WW/` (excluding `Weekly Review.md` and `Weekly Todo.md`). For each note, extract:
   - Completed action items (`[x]`)
   - Carried-forward items (`[>]`)
   - The `## Plan > ### This week` block — these are week-scope intentions the user wrote

3. **Read the Weekly Todo** — Read `01_Execution/YYYY-[W]WW/Weekly Todo.md`. For structured task status, use `obsidian tasks done path="01_Execution/YYYY-WXX/Weekly Todo.md"` and `obsidian tasks todo path="01_Execution/YYYY-WXX/Weekly Todo.md"` to get done/open lists. Fallback: read the file directly with Read tool.
   - `[x]` → done ✅
   - `[ ]` → not done ❌
   - Use this to populate `### Week Intentions Captured` with accurate ✅ / ❌ / ⬜ status.
   - Cross-reference against `### This week` items from daily notes to catch anything not in the Weekly Todo.

3b. **Read Blockers** — Read `01_Execution/YYYY-[W]WW/Blockers.md` if it exists.
   - For each `## Waiting On` item: record `[x]` (delivered) or `[ ]` (still blocking).
   - For each `## Meetings` entry: record `[x]` (happened) or `[ ]` (missed/cancelled).
   - Include undelivered blocker items in `### Stalled` section as blocking dependencies.
   - Include meeting outcomes in `### Progress` if relevant.

4. **Read active projects** — Read all active projects in `02_Projects/` (status: active). Note recent changes and stalled threads.

5. **Synthesize** — Based on the above, produce content for the `## AI Synthesis` section covering:
   - **Progress** — what actually moved forward across projects and daily notes
   - **Stalled** — what didn't move and why (if visible from the notes)
   - **Patterns & risks** — themes emerging across projects worth watching
   - **Week Intentions Captured** — each committed todo item from Weekly Todo with ✅ (done) / ❌ (not done) / ⬜ (partial); supplement with `### This week` items from daily notes not captured in the Weekly Todo
   - **Next week's focus** — top 3–5 priorities and any decisions needed

5b. **Detect horizon items** — Semantically scan all daily notes read in step 2 for items that belong at a higher planning horizon. Look for:
   - Future month mentions ("March", "三月", "next month", specific month names in any language)
   - Quarter references ("Q2", "next quarter", "下个季度")
   - Deferral language ("记入月度计划中", "defer", "push to", "not now but", "later", "put aside", "下个月")
   - Deadlines without calendar events ("deadline Apr 9", "due date", "due Mar 15", "截止")
   - Any item semantically indicating "not this week, but later"

   For each detected item, record: the item text, which horizon it belongs to (month/quarter), and the source daily note date. If no items are detected, omit the `### Horizon Items` section entirely.

6. **Create the weekly note** — Write `01_Execution/YYYY-[W]WW/Weekly Review.md` using the structure below. Populate `## AI Synthesis` with your synthesis. Leave `## Reflection` and `## Todos next week` blank for manual fill-in. If the file already exists and contains a `### AI Weekly Digest` block (inserted by `/ai-weekly-digest`), preserve that block in the output. (The week folder should already exist from daily notes or `/weekly-init`; do not attempt to create it.)

7. **Open the file** — Run `obsidian open path="01_Execution/YYYY-WXX/Weekly Review.md"` to open the file in Obsidian for the user to complete.

## Output File Structure

```markdown
---
type: weekly-review
date: YYYY-MM-DD
week: YYYY-[W]WW
status: active
---

# Weekly Review · YYYY-[W]WW

## AI Synthesis

### Progress
[What actually moved forward this week]

### Stalled
[What didn't move and why, if visible]

### Patterns & Risks
[Themes emerging across projects]

### Week Intentions Captured
[Each todo from Weekly Todo — ✅ done / ❌ not done / ⬜ partial]
[Items from ### This week in daily notes not in Weekly Todo — marked ✅/❌/⬜]

### Next Week's Focus
[Top 3–5 priorities and decisions needed]

### Horizon Items
[Items detected from daily notes that belong at monthly/quarterly horizon]
[Each item with: text, suggested horizon (month/quarter), source date]
[Omit this section entirely if nothing detected]

## Reflection

> Fill in manually.

## Todos next week

- [ ]
- [ ]
- [ ]
```

Claude populates `## Todos next week` with AI-suggested priorities. The user can promote these items into next week's Weekly Todo when `/weekly-init` runs.

Keep the synthesis tight and signal-dense. Surface only what's decision-relevant or pattern-forming — no padding.