---
name: weekly-init
description: "Invoke IMMEDIATELY when user wants to start a new week or says /weekly-init. Triggers on any request to set up, initialize, or kick off a week — including Monday morning setup, weekly planning, creating W## folders, building a weekly todo or blockers file, carrying over unfinished items, or pulling in deadline tasks. If the user mentions 'new week', 'start the week', 'get organized for this week', 'execution layer', or wants to carry forward from last week, this is the skill. Not for: weekly review/retrospective, daily briefing, calendar lookup, deadline plan creation, or progress tracking."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, weekly-planning, execution, carry-forward]
---

Initialize the current week: create the week folder, carry unfinished items from last week, and populate the Weekly Todo.

**CLI fallback:** If any `obsidian` CLI command fails, silently use the equivalent file tool (Read, Write, Edit). Do not surface CLI errors to the user.

## Steps

1. **Compute the current week string** — Use today's date to derive `YYYY-[W]WW` (e.g. `2026-W08`). Also compute last week's string (`YYYY-[W](WW-1)`), handling year boundaries correctly.

2. **Create the week folder** — Ensure `01_Execution/YYYY-[W]WW/` exists. No-op if it already exists.

3. **Check for existing Weekly Todo** — If `01_Execution/YYYY-[W]WW/Weekly Todo.md` already exists, stop and inform the user: "Week already initialized — Weekly Todo already exists at `01_Execution/YYYY-[W]WW/Weekly Todo.md`." Do not overwrite.

4. **Carry items from last week** — Two sources, both go into `### Unscheduled`:

   **a. Last week's Weekly Review `## Todos next week`** — Look for `01_Execution/YYYY-[W](WW-1)/Weekly Review.md`.
   - If found: extract every `- [ ]` line under the `## Todos next week` section.
   - These are the primary carry source — they represent AI-synthesised + user-reviewed priorities.
   - Do NOT append `(carried)` — these are fresh intentional commitments, not leftovers.

   **b. Last week's Weekly Todo uncompleted items** — Look for `01_Execution/YYYY-[W](WW-1)/Weekly Todo.md`.
   - If found: extract every `- [ ]` line from all sections that has actual text (skip blank `- [ ]` placeholders).
   - Strip any day prefix (`(Mon)`, `(Tue)`, etc.).
   - Append `(carried)` as a suffix to each item's text, e.g. `- [ ] finish project spec (carried)`.
   - Deduplicate against items already pulled from the Weekly Review.

   If neither file exists: no carried items.

5. **Parse this-week items from existing daily notes** — Read all `.md` files in `01_Execution/YYYY-[W]WW/` that are not `Weekly Review.md` or `Weekly Todo.md`.
   - From each note, extract the `## Plan > ### This week` block (all bullet items under `### This week` within `## Plan`).
   - For each item, strip any day prefix `(Mon)`, `(Tue)`, etc. before adding to the flat list.
   - Deduplicate across notes by exact text match (after stripping the day prefix).
   - Do NOT include items already in carried items (deduplicate against those too).

5b. **Inject deadline tasks** — Glob for all `02_Projects/**/Deadline Plan.md` (recursive). For each active (non-✅) deadline:
   - Read the `## Task Queue` section for that deadline — collect all pending `[ ]` tasks in order.
   - Pull the **top 3–5 tasks** from the queue into Weekly Todo (simple count, not hour-based filling). Add each as:
     `- [ ] [<path>] <task description>`
     e.g. `- [ ] [ProjectA/SubjectX] Revise module 3–5`
     e.g. `- [ ] [ProjectA/SubjectX] Past paper 1 + corrections`
   - Hours tracking stays in the deadline plan (not duplicated in Weekly Todo items).
   - If the queue has fewer than 3 pending tasks, add a planning item:
     `- [ ] [<path>] Plan deadline tasks (queue low)`
   - Deduplicate against carried/parsed items — if a matching task already exists (exact text match), skip it.

6. **Write `01_Execution/YYYY-[W]WW/Weekly Todo.md`** using the structure below.
   - Write all carried/parsed items as a flat checklist directly under the title — no day sections.
   - If there are no items at all, leave a single empty `- [ ]` placeholder.

7. **Create Blockers.md** — Check if `01_Execution/YYYY-[W]WW/Blockers.md` exists. If not, create it with standard frontmatter (`type: blockers`, `week: YYYY-[W]WW`, `date: YYYY-MM-DD`, `status: active`) and empty `## Waiting On` and `## Meetings` sections. If last week's `01_Execution/YYYY-[W](WW-1)/Blockers.md` exists:
   - **`## Waiting On`**: carry forward any `[ ]` items, appending `(carried)` suffix to each. Do not carry `[x]` items.
   - **`## Meetings`**: carry forward `[ ]` meeting entries whose parsed date falls **within or after the new week** (i.e. rescheduled meetings). Parse the date from the bold text (e.g. `**Sat Mar 7, 8 PM**`). Carry the entry with its full sub-bullet agenda intact, and append `(rescheduled)` suffix to the top-level entry text. Discard: past-dated `[ ]` meetings (date before the new week's Monday), `[-]` (cancelled), and `[x]` (completed).

7b. **Populate `## Meetings` from Weekly Review + carried items** — Scan the sources already read in Steps 4–5 for items containing a **date falling within the new week** (Mon–Sun):

   **Sources:**
   - Weekly Review `### Next Week's Focus` — items mentioning specific dates (e.g. `NLV call (Mar 4)`, `BP deadline Tue`)
   - Weekly Review `## Todos next week` — items with date-stamped parentheticals (e.g. `(call is Wed Mar 4)`, `(deadline ~Mar 4)`)
   - Carried items from last week's Weekly Todo — items with embedded dates

   **Date detection:** Match patterns like day names (`Mon`/`Tue`/.../`Sun`), month-day (`Mar 4`, `March 4`, `3/4`), day-of-week names (`Monday`, `Wednesday`), or `~Mar N` within item text (including parentheticals and suffixes). Resolve each to a concrete date within the new week.

   **For each extracted event with a date in the new week:**
   - **Skip deadlines** — items like `(deadline ~Mar 4)`, `NLP assignment due Mar 4`, `submission deadline` are NOT meetings. They belong in Weekly Todo / Deadline Plans. Only extract items that represent **scheduled interactions**: calls, meetings, pitches, interviews, presentations, demos, syncs.
   - **Skip duplicates** — if the event is already in `## Meetings` (from carry-forward in Step 7), skip it. Deduplicate by date + fuzzy title match.
   - **Distinguish event from prep task:**
     - If the item describes the event itself (e.g. `NLV call (Mar 4)`) → create a Blockers `## Meetings` entry.
     - If the item describes prep for the event (e.g. `NLV call prep — narrative, demo script (call is Wed Mar 4)`) → extract the event from the parenthetical, create a Blockers `## Meetings` entry for the event, keep the prep task in Weekly Todo as-is.
   - **Entry format:** `- [ ] **Day Mon DD** — Event name — [ProjectName] [[Weekly Review · YYYY-WXX]]`
     - e.g. `- [ ] **Wed Mar 4** — NLV Zoom call — [FM-Copilot] [[Weekly Review · 2026-W09]]`
     - Infer the project name from context (the event name, the weekly review source, or the project the meeting relates to). If no project can be inferred, use `[General]`.
     - If a time is mentioned in the source text, include it: `**Wed Mar 4, 3:30 PM**`
     - If participants are mentioned, include them as a sub-bullet.

7c. **Populate `## Meetings` from ICS calendar** — Read `.obsidian/plugins/obsidian-day-planner/data.json` and parse the `rawIcals` array. For each calendar entry, extract `VEVENT` blocks from the `text` field whose `DTSTART` falls within the new week (Mon–Sun).

   For each matching event, extract:
   - Title from `SUMMARY`
   - Start/end time from `DTSTART`/`DTEND` (convert to local timezone)
   - Description from `DESCRIPTION` (if present)
   - Calendar source from `icalConfig.name`

   **Filtering — skip these:**
   - **Recurring academic lectures/tutorials**: has `RRULE` AND course-code-like SUMMARY (e.g. `COMP70050`, `MATH40002`). Exception: if a deadline plan exists for that subject AND a deadline is within 14 days, keep the event as context.
   - **All-day events**: `DTSTART` value has no `T` component (date-only, e.g. `20260304` not `20260304T140000`).
   - **Already in `## Meetings`**: deduplicate by date + fuzzy title match against entries from Step 7 carry-forward and Step 7b.

   **Keep:** one-off meetings, interviews, calls, pitches, presentations, exams, office hours — anything representing a scheduled interaction or assessment.

   **Entry format:** `- [ ] **Day Mon DD, H:MM PM** — Event title — [calendar source]`
   - e.g. `- [ ] **Thu Mar 6, 2:00 PM** — Interview with Company — [ProjectA]`
   - Include description/location as a sub-bullet if available.

   **Fallback:** If `data.json` is unreadable or `rawIcals` is empty, skip silently — do not error.

7d. **Populate `## Meetings` from Upcoming Events files** — Glob for all `02_Projects/**/Upcoming Events.md` files (recursive). For each file:

   - Read `## Events` section and find all `[ ]` entries whose parsed date falls within the new week (Mon–Sun).
   - Parse the date from the bold text (e.g. `**Thu Apr 9, 5:00 PM**`).
   - For each matching event:
     - Add to Blockers `## Meetings` using the same entry format, with source attribution `[<project>]`.
     - Carry sub-bullets (location, notes, etc.) into the Blockers entry.
     - Mark the event in the Upcoming Events file: replace `- [ ]` with `- [x] consumed YYYY-WXX:` on that line.
   - **Skip duplicates** — deduplicate by date + fuzzy title match against entries from Steps 7, 7b, and 7c.
   - **Skip past events** — if the event date is before the new week's Monday and still `[ ]`, flag it in the console: "Stale event in <file>: <event name> (<date>)". Do not consume or carry it.

   **Fallback:** If no Upcoming Events files exist, skip silently.

8. **Open the file** — Run `obsidian open path="01_Execution/YYYY-WXX/Weekly Todo.md"` to open the file in Obsidian so the user can review and adjust before the week begins.

## Output File Structure

```markdown
---
type: weekly-todo
week: YYYY-[W]WW
date: YYYY-MM-DD
status: active
---

# Weekly Plan · WXX · YYYY

- [ ] <item>
- [ ] <item>
```

Flat list only — no day sections. `/daily-init` surfaces all uncompleted items each day and syncs completions back automatically.

## Notes

- This command is also auto-triggered by `/daily-init` when it detects no Weekly Todo for the current week.
- After `/weekly-init` runs, the user should open the Weekly Todo and add or remove items before starting work.
- Do not modify last week's Weekly Todo — it is a historical record.