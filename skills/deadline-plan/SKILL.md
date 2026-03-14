---
name: deadline-plan
description: "Use when the user wants to create, view, or manage a deadline-driven work plan. This skill backward-schedules hours across weeks leading to a due date — ramping intensity for exams, steady pacing for coursework/assignments, or front-loaded effort for project milestones and demos. Invoke for: creating a deadline plan or study schedule for exams, papers, assignments, project phases, or MVP demos; checking deadline progress (hours done vs expected across plans); scheduling exam prep or coursework with total hour targets distributed over weeks; any /deadline-plan command. NOT for: adding single calendar events, daily briefings, weekly reviews, time-tracking lookups, or annual goal setting."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, deadline, scheduling, ramp, progress-tracking]
---

Backward-schedule hard deadlines (exams, coursework, project milestones) across weeks with automatic progress tracking.

## Arguments

`/deadline-plan <mode> [path]`

- `init <path>` — Create a new deadline plan at the given path
- `view [path]` — Show status of all deadline plans (or one specific plan)

`<path>` is a relative path under `02_Projects/` — can be a top-level project or any subfolder:
- `ProjectA/SubjectX` → `02_Projects/ProjectA/SubjectX/Deadline Plan.md`
- `ProjectB/Phase1` → `02_Projects/ProjectB/Phase1/Deadline Plan.md`
- `ProjectC` → `02_Projects/ProjectC/Deadline Plan.md`

If no mode is given, infer from context:
- If `<path>` has no `Deadline Plan.md` → `init`
- If no path given → `view`

Note: There is no `update` mode. All syncing (marking tasks done, updating hours, rebalancing) is handled automatically by `/daily-init` Step 0b.

---

## Mode: `init <path>`

1. **Validate** — Confirm `02_Projects/<path>/` exists. If `02_Projects/<path>/Deadline Plan.md` already exists, stop: "Deadline plan already exists. Run `/deadline-plan update <path>` to refresh."

2. **Gather deadlines** — Ask the user interactively:
   > List your deadlines for <path>. For each, provide:
   > - Name (e.g. "NLP Coursework", "ML Exam", "Demo MVP")
   > - Type: `coursework`, `exam`, or `project`
   > - Due date (YYYY-MM-DD)
   > - Estimated total hours to complete
   > - (Optional) Start date or "after <other deadline name>" — defaults to now
   >
   > Example: `NLP Coursework, coursework, 2026-04-15, 40`
   > Example with start: `Essay, coursework, 2026-04-16, 6, after Presentation`
   > Example with date: `Essay, coursework, 2026-04-16, 6, start 2026-03-20`

   Wait for the user's response. Parse each line into `{name, type, date, hours_total, start}`. If the user omits type, infer from the name (keywords like "exam", "test", "midterm", "final" → `exam`; "coursework", "assignment", "report", "essay" → `coursework`; everything else → `project`).

   **Start date resolution:**
   - If omitted → starts from current week (default)
   - If `after <name>` → resolves to the due date of the referenced deadline within the same plan
   - If `start YYYY-MM-DD` → uses that explicit date
   - The start date determines the first week that receives hours (see Start-week fraction in Ramp Algorithm)

3. **Auto-detection keywords** — Ask:
   > What keywords should I use to detect related work in daily notes?
   > Defaults: each path segment + each deadline name. Add any extras?

   Parse response. Default: each segment of the path (e.g. `ProjectA/SubjectX` → `ProjectA`, `SubjectX`) + each deadline name (lowercased).

3b. **Gather initial tasks** — Ask:
   > For each deadline, list any tasks you already know. Format: `task description, hours`
   > Leave blank to add later. AI will suggest tasks based on your deadline type and phase.

   Wait for the user's response. For each deadline:
   - If the user provides tasks, parse them into `{description, hours}` and write into the Task Queue as `- [ ] <description> — <hours>h`.
   - If the user skips (blank), AI generates 3–5 starter tasks based on the deadline type:
     - `exam` → "First pass through lecture notes", "Practice problem set", "Past paper + corrections", "Revise weak topics", "Final review"
     - `coursework` → "Read brief / requirements", "Research + outline", "Draft section 1", "Review + polish", "Final check + submit"
     - `project` → "Scope + plan", "Build core feature", "Test + iterate", "Documentation", "Final delivery"
   - Starter tasks get rough hour estimates: distribute `hours_total` across the tasks (roughly equal, rounded to 0.5h).

4. **Compute ramp schedule** — For each deadline:
   - Determine the first active week: the start date's ISO week (or current week if no start date).
   - Count weeks from the first active week to the deadline's ISO week (inclusive). Call this N.
   - Compute the start-week fraction and deadline-week fraction (see Ramp Algorithm below).
   - Apply the ramp algorithm to distribute `hours_total` across N weeks, scaling the first and last weeks by their fractions.
   - Record per-week targets. Weeks before the start week get `—` (not 0h).

5. **Write `02_Projects/<path>/Deadline Plan.md`** — Use the output format below.

6. **Apple Calendar events** — Ask for confirmation, then create all-day calendar events for each deadline date. Use the calendar name from CLAUDE.md Customization table. **Date format must be `"D Month YYYY"` (e.g. `"19 March 2026"`) — AppleScript does NOT parse `YYYY-MM-DD`.**
   ```
   osascript -e 'tell application "Calendar" to tell calendar "<calendar_name>" to make new event with properties {summary:"DEADLINE: <name> (<path>)", start date:date "<D Month YYYY>", end date:date "<D Month YYYY>", allday event:true}'
   ```

7. **Apple Reminders** — Ask for confirmation, then create reminders 7 days before each deadline. **Same date format: `"D Month YYYY"`.** Use the reminders list name from CLAUDE.md Customization table.
   ```
   osascript -e 'tell application "Reminders" to tell list "<reminders_list>" to make new reminder with properties {name:"1 week until: <name> (<path>)", due date:date "<D Month YYYY>"}'
   ```

8. **Open in Obsidian** — `obsidian open path="02_Projects/<path>/Deadline Plan.md"`

---

## Mode: `view [path]`

1. **Find plans** — If `<path>` given, read `02_Projects/<path>/Deadline Plan.md`. Otherwise, glob `02_Projects/**/Deadline Plan.md` for all plans (recursive — catches nested paths).

2. **Render status table** — Output to the conversation (not a file). The "Path" column shows the relative path under `02_Projects/`:

   ```
   ## Deadline Status

   | Path | Deadline | Type | Date | Days Left | Done/Total | Status |
   |------|----------|------|------|-----------|------------|--------|
   | ProjectA/SubjectX | Coursework | coursework | 2026-04-15 | 45 | 12.0/40h | 🟢 |
   | ProjectA/SubjectY | Final Exam | exam | 2026-06-01 | 92 | 0.0/30h | 🟡 |
   | ProjectB/Phase1 | Demo MVP | project | 2026-05-01 | 61 | 5.0/25h | 🟢 |
   ```

3. **Flag risks** — Below the table, list any deadlines that are:
   - Within 14 days
   - Status 🔴
   - Status 🟡 with < 21 days remaining

---

## Ramp Algorithm

Given `hours_total`, `N` weeks (whole weeks — if a deadline starts mid-week, that week counts as week 1), and `type`:

### `after <name>` resolution

When deadline B says `after A`, B's start date = A's due date. B's first week = the ISO week containing A's due date. Weeks before the start week show `—` (not allocated).

### Distribution by type

Use discrete multipliers per week. Divide weeks into three equal(ish) thirds: early, middle, final.

**`exam`** (back-loaded — increasing intensity):
- Early weeks: each week gets `0.5 × base`
- Middle weeks: each week gets `1.0 × base`
- Final weeks: each week gets `1.5 × base`

**`project`** (front-loaded — planning + delivery ramp):
- Early weeks: each week gets `1.5 × base`
- Middle weeks: each week gets `1.0 × base`
- Final weeks: each week gets `0.5 × base`

**`coursework`** (flat — steady effort):
- All weeks: each week gets `1.0 × base`

Where `base = hours_total / sum_of_all_multipliers`. This ensures total hours are preserved.

### Short deadlines
- **N = 1**: Week 1 gets 100% of hours
- **N = 2–3**: Use flat distribution (all weeks × 1.0) regardless of type

### Common rules
- Apply minimum floor: 2.0h/week per deadline. If any week falls below floor, pull surplus from the latest weeks (working backwards).
- Round all values to 1 decimal place.

When recomputing after progress, apply the same algorithm to `remaining` hours across the current week and all future weeks. Weeks before the current week retain their original targets. The type determines which profile is used for rebalancing.

---

## Status Assessment

```
expected_done = sum of weekly targets for all past weeks (weeks before current week)
pace = hours_done / expected_done   (if expected_done == 0, pace = 1.0)

🟢  pace >= 1.0
🟡  pace 0.75–0.99
🔴  pace < 0.75, OR (deadline <= 7 days away AND remaining > 4h)
✅  hours_done >= hours_total
```

---

## Output Format: `02_Projects/<path>/Deadline Plan.md`

```markdown
---
type: deadline-plan
path: <path>
date: YYYY-MM-DD
last-updated: YYYY-MM-DD
keywords: [<segment1>, <segment2>, <deadline1>, <deadline2>, ...]
---

# Deadline Plan · <path>

## Deadlines
| Name | Type | Date | Start | Total | Done | Remaining | Status |
|------|------|------|-------|-------|------|-----------|--------|
| NLP Coursework | coursework | 2026-04-15 | — | 40.0h | 0.0h | 40.0h | 🟢 |
| Essay | coursework | 2026-04-16 | after Presentation | 6.0h | 0.0h | 6.0h | 🟢 |

## Weekly Allocation
| Week | Week Of | NLP Coursework | ML Exam | Total |
|------|---------|----------------|---------|-------|
| W10 | Mar 2 | 2.0h | 2.0h | 4.0h |
| W11 | Mar 9 | 2.5h | 2.0h | 4.5h |
| ... | ... | ... | ... | ... |

## Per-Deadline Breakdown

### NLP Coursework
| Week | Target | Phase |
|------|--------|-------|
| W10 | 2.0h | Light |
| W11 | 2.5h | Light |
| ... | ... | ... |

### ML Exam
| Week | Target | Phase |
|------|--------|-------|
| W10 | 2.0h | Light |
| ... | ... | ... |

## Task Queue

### NLP Coursework
- [ ] Revise module 1–3 — 2.0h
- [ ] Revise module 4–6 — 2.0h
- [ ] Past paper 1 + corrections — 2.0h

### ML Exam
- [ ] First pass through lecture notes — 3.0h
- [ ] Practice problem set 1 — 2.0h

## Progress Log
_(empty until first sync)_

## Auto-Detection Rules
**Keywords:** <project>, <deadline1>, <deadline2>, ...

**Time extraction:** Matches `[x]` items in daily notes containing any keyword. Duration from Day Planner `HH:MM - HH:MM` format; defaults to 1.0h if unclocked.
```

---

## Task Queue Format

Each task in the queue uses one of two formats:
- **Pending:** `- [ ] <description> — <N.0>h`
- **Done:** `- [x] <YYYY-MM-DD>: <description> — <N.0>h`

Tasks are grouped by deadline name under `## Task Queue`. `/weekly-init` pulls the top 3–5 pending tasks into the Weekly Todo; `/daily-init` Step 0b marks them done and updates hours when the user completes them.

Users can add tasks to the queue at any time by editing the Deadline Plan directly.

## Notes

- This command is also read by `/daily-init` (Step 0b + 3d) and `/weekly-init` (Step 5b) to sync completions and surface deadline work.
- There is no `update` mode — all syncing is automatic via `/daily-init` Step 0b.
- The ramp algorithm front-loads review/buffer time at the end to avoid last-minute cramming.
