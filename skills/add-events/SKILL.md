---
name: add-events
description: "Batch-add events to Apple Calendar and Reminders with automatic routing into the Obsidian execution layer. TRIGGER when the user wants to add events, meetings, or appointments to their calendar — whether from pasted text, bullet lists, or described events. Signal phrases: 'add these events', 'put this on my calendar', 'schedule these meetings', 'add events for project X', 'create calendar events from this list'. Also triggers for /add-events. Creates Apple Calendar events, Apple Reminders for deadlines, writes Upcoming Events file, and routes current-week events to Blockers.md. NOT for viewing calendar, daily briefing, meeting prep, or deadline planning."
version: 1.0.0
author: Yuhan Wang
---

Batch-add events to Apple Calendar ("Operator") and associated deadlines to Apple Reminders, with automatic routing into the Operator execution layer.

**CLI fallback:** If any `obsidian` CLI command fails, silently use the equivalent file tool (Read, Write, Edit). Do not surface CLI errors to the user.

## Arguments

- `/add-events [project]` — project name optional, inferred from conversation context if possible
- Event data provided interactively (pasted text, structured list, screenshot description)
- Examples: `/add-events Millennium`, `/add-events`

## Steps

### Step 1 — Gather event data

- Prompt user to paste/describe events if not already provided
- Accept any format: free text, bullet lists, tables, screenshot descriptions
- For each event, extract: name, date, time+duration (optional), location/link (optional), associated deadlines (optional), notes (optional)

### Step 2 — Parse events

Extract structured fields for each event:

| Field | Required | Notes |
|-------|----------|-------|
| `name` | yes | Event title |
| `date` | yes | Resolve any format; relative dates against today; assume next occurrence if year omitted |
| `time_start` | no | If omitted, treat as all-day |
| `time_end` | no | Default: `time_start + 60min` if only start given |
| `location` | no | Physical address or video link |
| `description` | no | Notes, access codes, agenda |
| `project` | no | From argument or inferred |
| `all_day` | auto | `true` if no time given |
| `deadlines[]` | no | Associated deadlines: "survey due", "RSVP", "registration closes" patterns |

### Step 3 — Confirm with user

Display parsed events as a table:

| Date | Time | Event | Location | Project |
|------|------|-------|----------|---------|

Display associated deadlines as a separate table:

| Deadline | Related Event | Due Date |
|----------|---------------|----------|

Show current-week routing preview: which events (if any) will be added to this week's Blockers.md.

Wait for user confirmation or corrections before proceeding.

### Step 4 — Create Apple Calendar events

Use osascript to create events in the "Operator" calendar. For each event:

```applescript
osascript -e '
tell application "Calendar"
  tell calendar "Operator"
    make new event with properties {summary:"EVENT_NAME", start date:date "D Month YYYY HH:MM:SS", end date:date "D Month YYYY HH:MM:SS", description:"DESCRIPTION", location:"LOCATION"}
  end tell
end tell'
```

- **Date format:** `"D Month YYYY HH:MM:SS"` for timed events, use `allday event` property for all-day
- **Description field:** concatenate link, access code, notes, deadlines, project name
- **Escape single quotes:** replace `'` with `'"'"'` in all text fields
- Execute sequentially, log any failures, continue with remaining events

### Step 5 — Create Apple Reminders

Use osascript to create reminders in the "Operator" list. For each deadline:

```applescript
osascript -e '
tell application "Reminders"
  tell list "Operator"
    make new reminder with properties {name:"DEADLINE_NAME — EVENT_NAME", body:"Related: EVENT_NAME on EVENT_DATE", due date:date "D Month YYYY HH:MM:SS"}
  end tell
end tell'
```

- **Name format:** `"<deadline name> — <event name>"`
- **Body:** related event name + date
- Skip this step if no deadlines were extracted

### Step 6 — Write Upcoming Events file

**Path:** `02_Projects/<project>/Upcoming Events.md`

- If the file doesn't exist, create it with frontmatter:
  ```yaml
  ---
  type: events
  project: <project>
  last-updated: YYYY-MM-DD
  ---

  # Upcoming Events · <Project>

  ## Events
  ```
- If the file exists, read it and append new events under `## Events`
- **Deduplicate:** skip if same date + fuzzy title match already exists as a `[ ]` entry
- Update `last-updated` in frontmatter to today

**Entry format** (matches Blockers convention):
```markdown
- [ ] **Day Mon DD, H:MM PM** — Event name — 60min
  - Location/link details
  - Access code, notes
  - Survey deadline: Day Mon DD, H:MM PM
```

For all-day events:
```markdown
- [ ] **Day Mon DD** — Event name — all day
  - Location/link details
```

- Events without a project → save to `02_Projects/General/Upcoming Events.md` (create project folder if needed)

### Step 7 — Route current-week events to Blockers.md

- Determine current ISO week boundaries (Monday–Sunday)
- For each event whose date falls within the current week:
  1. Read or create `01_Execution/YYYY-WXX/Blockers.md` (with standard frontmatter + empty `## Waiting On` and `## Meetings` sections if new)
  2. **Deduplicate** against existing `## Meetings` entries by date + fuzzy title match
  3. Append to `## Meetings` in chronological order, using the same entry format as Step 6 with source attribution `[<project>]`
  4. Use obsidian CLI (`obsidian append`) with Edit tool fallback
- Mark consumed events in `Upcoming Events.md`: replace `- [ ]` with `- [x] consumed YYYY-WXX:` on that line

### Step 8 — Report summary

Output a summary:

- Apple Calendar events created: N
- Apple Reminders created: N
- Upcoming Events file: `02_Projects/<project>/Upcoming Events.md`
- Current-week Blockers entries: N (or "none — all events are in future weeks")
- Pipeline note: "Future events will be auto-populated into Blockers by `/weekly-init`"
- If project linked: suggest running `/meeting-prep` for upcoming meetings

## Notes

- **Past events:** warn the user if any parsed event date is in the past; still create if user confirms
- **Timezones:** convert to local timezone; if source specifies a different zone, note both in the description field
- **Events without a project:** route to `02_Projects/General/Upcoming Events.md` (create folder if needed)
- **`/weekly-init` Step 7d** reads Upcoming Events files — no manual intervention needed for future weeks
- This command is standalone — no auto-triggers. Run it whenever you have events to add.
