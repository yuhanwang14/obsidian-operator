---
name: quarterly-plan
description: "Use when the user wants to check on quarterly goals, assess objective status (on track/at risk/off track), initialize a new quarter's plan, review a completed quarter's results, or run a monthly pulse checkpoint. Triggers for: tracking quarterly goal progress, mid-quarter checkpoints, quarter-end retrospectives with objective-by-objective analysis, setting up a new quarter (e.g. 'set up Q2'), monthly pulse reports showing how goals are progressing, and horizon item tracking. Also handles the /quarterly-plan slash command with subcommands like init, review, and pulse. Do NOT use for annual vision/review, weekly planning, general strategy summaries, or meeting prep."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, strategy, quarterly, planning, goals]
---

Strategic quarterly planning: init, review, and monthly pulse checkpoint.

## Arguments

- `pulse [MM]` — Run monthly pulse for month MM (default: last month). Auto-triggered by `/daily-init` on 1st of each month.
- `init [YYYY-QX]` — Initialize a new quarter's plan. Auto-triggered by `/daily-init` on first Monday of quarter.
- `review [YYYY-QX]` — Review a completed quarter. Auto-triggered by `/daily-init` on first Monday of new quarter.

If no argument given, auto-detect mode:
- If current quarter has no `Quarterly Plan.md` → init
- If last quarter has no `Quarterly Review.md` → review
- Otherwise → pulse for current month

## Pulse Mode (Monthly Checkpoint)

### Step 1: Determine context
Compute current quarter string `YYYY-QX`. Determine which month within the quarter (1st, 2nd, or 3rd). Default month = last month if not specified.

### Step 2: Read sources
- All Weekly Reviews from the target month in `01_Execution/YYYY-WXX/Weekly Review.md` — especially `### Horizon Items` (under `## AI Synthesis`) and `## Reflection` sections (surface reflection themes in qualitative assessment)
- Current `00_Strategy/YYYY-QX/Quarterly Plan.md` — the goals being tracked
- Active project notes in `02_Projects/` (status: active) — recent progress, `## Now`, `## Risks`, and `## Strategic Signals` sections
- Knowledge notes created during the target month in `04_Knowledge/` — meetings, decisions, research, brainstorms

### Step 3: Assess quarterly goals
For each objective/milestone in the Quarterly Plan, assess status:
- 🟢 on track — clear progress, no blockers
- 🟡 at risk — some progress but behind schedule or blocked
- 🔴 off track — no progress or major blockers

When assessing each goal, incorporate signals from project-level research (knowledge notes and `## Strategic Signals` from project notes). If recent research reinforces, challenges, or introduces new considerations for a goal, note it inline with the assessment — e.g. "🟡 at risk — competitive landscape shifted per [[Relevant Knowledge Note]]".

### Step 4: Collect horizon items
Aggregate all `### Horizon Items` from weekly reviews in the target month. Deduplicate by semantic meaning.

### Step 5: Identify maturing deadlines
From collected horizon items and project notes, identify items that now have concrete dates. Classify each:
- **Has exact date + time** → offer to create Apple Reminder (via osascript, using the reminders list name from CLAUDE.md)
- **Has exact date only** → offer to create Apple Calendar all-day event (via osascript, using the calendar name from CLAUDE.md)
- **Still soft** → keep in vault planning layer

For each item to be created, present to user and confirm before running osascript:
```applescript
-- Apple Calendar all-day event (use calendar name from CLAUDE.md Customization)
tell application "Calendar"
    tell calendar "<calendar_name>"
        make new event with properties {summary:"DEADLINE: <item>", start date:date "<YYYY-MM-DD>", allday event:true}
    end tell
end tell

-- Apple Reminder with exact time (use reminders list from CLAUDE.md Customization)
tell application "Reminders"
    tell list "<reminders_list>"
        make new reminder with properties {name:"<item>", due date:date "<YYYY-MM-DD HH:MM:SS>"}
    end tell
end tell
```

### Step 6: Write Monthly Pulse
Write `00_Strategy/YYYY-QX/Monthly Pulse - MM.md` using the template structure. Populate all sections from the analysis above.

### Step 7: Open the file
Run `obsidian open path="00_Strategy/YYYY-QX/Monthly Pulse - MM.md"` to open the file in Obsidian.

## Init Mode (Start of Quarter)

### Step 1: Determine quarter
Compute current quarter `YYYY-QX`. Create folder `00_Strategy/YYYY-QX/` if it doesn't exist.

### Step 2: Guard
If `00_Strategy/YYYY-QX/Quarterly Plan.md` already exists, stop: "Quarter already initialized."

### Step 3: Read sources
- `00_Strategy/YYYY Vision.md` — annual goals and themes (if exists)
- Last quarter's `Quarterly Review.md` — suggested focus + lessons (if exists)
- Last quarter's monthly pulses — trajectory and open items
- Active project notes in `02_Projects/`

### Step 4: Guide goal-setting
Present to user:
- Annual goals relevant to this quarter (from Vision)
- Suggested focus from last quarter's review
- Carried horizon items that weren't resolved
- Ask user to confirm/adjust objectives before writing

### Step 5: Write Quarterly Plan
Write `00_Strategy/YYYY-QX/Quarterly Plan.md` using the template. Populate objectives, milestones, key results, monthly focus, and risks from the discussion.

### Step 6: Open the file
Run `obsidian open path="00_Strategy/YYYY-QX/Quarterly Plan.md"` to open the file in Obsidian.

## Review Mode (End of Quarter)

### Step 1: Determine quarter
Default: last completed quarter. Compute `YYYY-QX`.

### Step 2: Guard
If `00_Strategy/YYYY-QX/Quarterly Review.md` already exists, stop: "Quarter already reviewed."

### Step 3: Read sources
- `00_Strategy/YYYY-QX/Quarterly Plan.md` — what was planned
- All monthly pulses for the quarter
- All weekly reviews for the quarter (from `01_Execution/YYYY-WXX/`) — including `## Reflection` sections for qualitative themes
- All active project notes in `02_Projects/`
- All knowledge notes created during the quarter in `04_Knowledge/`

### Step 4: Synthesize
Produce analysis:
- Objective-by-objective: achieved / partial / missed
- Wins and misses with reasons
- Patterns across the quarter (from monthly pulses + weekly reviews)
- Horizon items that were collected but unresolved → carry to next quarter
- Suggested next quarter focus

### Step 5: Write Quarterly Review
Write `00_Strategy/YYYY-QX/Quarterly Review.md` using the template. Leave space for manual reflection.

### Step 6: Open the file
Run `obsidian open path="00_Strategy/YYYY-QX/Quarterly Review.md"` to open the file in Obsidian.
