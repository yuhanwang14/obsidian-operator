---
name: daily-init
description: "Daily kickoff ritual — invoke for /daily-init, \"start my day\", \"morning briefing\", \"set up today's note\", or any request to begin, initialize, or plan the workday. Builds today's daily note by pulling Gmail, calendar, weekly todos, yesterday's carry-forwards, and active deadline plans into a structured briefing with action items and a time-blocked schedule. Optional args: hours budget, manual items (e.g. \"/daily-init 6, buy groceries\"). Auto-triggers week/month/quarter boundary reviews when needed."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, daily-planning, briefing, execution, day-planner]
---

Generate today's daily briefing by gathering data from all sources below, then output a single concise briefing.

**CLI fallback:** If any `obsidian` CLI command fails, silently use the equivalent file tool (Read, Write, Edit). Do not surface CLI errors to the user.

## Arguments

**First argument — daily budget (hours):** An optional number indicating how many productive hours the user has today. Examples: `6`, `4.5`, `8`.

- If provided, use this as the daily budget for scheduling.
- If NOT provided, **ask the user** before proceeding: "How many hours of productive work do you have today?" Wait for their answer before generating the briefing.

**Remaining arguments — manual action items:** A comma-separated list of extra action items to include.

Examples:
- `/daily-init 5` — 5h budget, no manual items
- `/daily-init 6, call dentist, review project deck` — 6h budget + manual items
- `/daily-init call dentist` — no budget (will prompt), one manual item
- `/daily-init` — no budget (will prompt), no manual items

Parsing rule: if the first argument is a bare number (integer or decimal), treat it as the daily budget. Everything else is parsed as comma-separated manual action items, tagged with "manual:" in Action Items.

## Pre-Flight Check

Run these checks **in order** before reading any other data source:

**Error handling:** If any auto-triggered step below fails (command errors or produces unexpected output), log a warning in `### Flags` (e.g. "⚠️ /weekly-review failed — run manually") and continue with the next step. Do not abort the entire briefing.

### 1. Close last week (new-week boundary)
Determine last week's ISO week number. Check whether `01_Execution/YYYY-[W]WW/Weekly Review.md` exists for **last week's** folder.
- If it does **not** exist and today is in a **different ISO week** from the most recent daily note → run `/weekly-review` for last week first, wait for it to complete, then continue.
- If it already exists → skip.

This fires on the first `/daily-init` of a new week, regardless of which day that is.

### 1b. AI Weekly Digest (new-week boundary)
After step 1, check whether `04_Knowledge/AI-Weekly/YYYY-[W]WW - AI Weekly Digest.md` exists for **last week**.
- If it does **not** exist and today is in a **different ISO week** from last week → run `/ai-weekly-digest`, wait for it to complete, then continue.
- If it already exists → skip.

### 1c. Monthly pulse (new-month boundary)
Check whether `00_Strategy/YYYY-QX/Monthly Pulse - MM.md` exists for **last month** (where MM is last month's two-digit number and YYYY-QX is last month's quarter).
- If it does **not** exist and today's month differs from last month → run `/quarterly-plan pulse` for last month, wait for completion.
- If it already exists → skip.

This fires on the first `/daily-init` of a new month, regardless of which day that is.

### 1d. Close last quarter (new-quarter boundary)
Check whether today's quarter differs from last quarter. If so, check whether `00_Strategy/YYYY-Q(X-1)/Quarterly Review.md` exists for **last quarter**.
- If it does **not** exist → run `/quarterly-plan review` for last quarter, wait for completion.
- If it already exists → skip.

This fires on the first `/daily-init` of a new quarter, regardless of which day that is.

### 1e. Open new quarter (new-quarter boundary)
Same trigger as 1d. Check whether `00_Strategy/YYYY-QX/Quarterly Plan.md` exists for the **current quarter**.
- If it does **not** exist → run `/quarterly-plan init`, wait for completion.
- If it already exists → skip.

### 2. Open this week
Check whether `01_Execution/YYYY-[W]WW/Weekly Todo.md` exists for the **current week**.
- If it does **not** exist → run `/weekly-init` first, wait for it to complete, then continue with the briefing.
- If it does exist → proceed directly to data sources.

This ensures Monday's `/daily-init` cleanly closes the old week before opening the new one.

## Data Sources

### 0. Sync Weekly Todo + Blockers from yesterday (skip if step 1 just ran /weekly-review)
Before reading other sources, open yesterday's daily note, the current Weekly Todo, and the current Blockers.md. Skip this step if `/weekly-review` was triggered in step 1 (the review already accounts for last week's completions).
- For each `[x]` item in yesterday's `### Action Items`, find any item in the Weekly Todo whose text closely matches (fuzzy — ignore prefixes like "carried:", "planned:", etc., compare the core task text).
- If a match is found and the Weekly Todo item is still `[ ]`, mark it `[x]`. Use `obsidian task done path="01_Execution/YYYY-WXX/Weekly Todo.md" line=N` (read the file first to get the line number). Fallback: Edit tool.
- Also check: if yesterday's `[x]` item semantically matches a Blockers.md `## Waiting On` `[ ]` item, mark the blocker `[x]` too (same CLI method, targeting Blockers.md).
- This keeps the Weekly Todo and Blockers in sync without the user having to update them manually.

**Sync cancelled meetings (`[-]`) from yesterday's daily note:**
- Scan yesterday's daily note for any `[-]` items (skipped/cancelled checkbox state).
- For each `[-]` item, check if it semantically matches a `## Meetings` entry in the current Blockers.md (match by meeting name, project keyword, or participant — e.g. "ProjectAlpha Meeting" matches a Blockers entry for "ProjectAlpha weekly").
- For each match where the Blockers entry is still `[ ]`: mark it `[-]` using Edit tool (replace `- [ ]` with `- [-]` on that line).
- If the daily note `[-]` item has sub-bullets with a reason (e.g. `今天没开，因为大家都没空`), append the reason as a sub-bullet under the Blockers `## Meetings` entry too.

### 0b. Sync Deadline Plans from yesterday
After syncing the Weekly Todo (Step 0), sync deadline plans. This replaces the old manual `update` mode — all sync is automatic. Skip this step if `/weekly-review` was triggered in step 1.

- Glob for all `02_Projects/**/Deadline Plan.md` files (recursive).
- For each `[x]` item in yesterday's daily note (both `### Action Items` and `## Schedule`):
  - Check if the item's text matches any deadline plan's keywords (case-insensitive substring match).
  - If matched, find the corresponding task in the deadline plan's `## Task Queue` under the matching deadline (fuzzy text match — compare core task description, ignoring prefixes like `[Project/Subject]`).
  - If a matching queue task is found and still `[ ]`:
    - Mark it `[x]` with yesterday's date: `- [x] YYYY-MM-DD: <description> — <N.0>h`
    - Add its hours to `hours_done` in the Deadlines table.
    - Recompute ramp: `remaining = hours_total - hours_done`, redistribute across the current week and all future weeks using the ramp algorithm (same profile as the deadline type). Weeks before the current week keep their original targets.
    - Update status using existing status assessment rules (🟢/🟡/🔴/✅).
    - Append to Progress Log: `- YYYY-MM-DD: <description> — N.0h`
  - If no matching queue task is found but the item clearly matches a deadline keyword, add the hours to `hours_done` anyway (use Day Planner duration if available, else 1.0h) and log it.
- Update `last-updated` in frontmatter to today's date.
- Rewrite the deadline plan file with all updates.

**Only recompute when hours changed:** If any tasks were marked done above (hours_done increased), recompute the weekly allocation:
- `remaining = hours_total - hours_done`
- Redistribute across the current week and all future weeks using the ramp algorithm (same profile as the deadline type).
- Weeks before the current week keep their original targets.
- Update the Weekly Allocation table and Per-Deadline Breakdown in the file.
- If no tasks were completed (hours unchanged), skip recomputation entirely.

**Queue health check:** After syncing, if any deadline's task queue has fewer pending hours than the next 2 weeks of targets combined, note this in `### Flags` (Step 3d) so the user knows to add more tasks.

### 1. Vault state
Read the most recently modified files in `01_Execution/` and any active projects in `02_Projects/` (status: active).

### 2. This week's daily notes
Read all daily note files in `01_Execution/YYYY-[W]WW/` (excluding `Weekly Review.md` and `Weekly Todo.md`).
   - From **yesterday's note**: extract:
     - Any action item marked `[>]` — check for an explicit delay date (text like "→ YYYY-MM-DD", "→ Mon", "delay to Mar 30", etc.):
       - **Delay date = today** → promote to a normal action item, prefixed "carried:" (the delay has landed)
       - **Delay date is in the future** → carry forward as `[>]` with the date preserved, prefixed "deferred:" (not yet actionable — passes through today's note visibly but stays `[>]`)
       - **No delay date** → treat as a regular carry-forward, prefixed "carried:" (same as before)
     - The `## Plan > ### Tomorrow` block — each bullet becomes a planned item
   - From **all this week's notes** (including yesterday): collect all `## Plan > ### This week` items:
     - Items with a (Day) prefix like (Mon), (Tue), etc. → surface only on that designated day
     - Items with **no day prefix** → surface starting the day after tomorrow (relative to today)
     - Deduplicate across notes by exact text match (after stripping day prefix)

### 3. Weekly Todo
Read `01_Execution/YYYY-[W]WW/Weekly Todo.md` (flat list — no day sections).
- Surface **all** `[ ]` items (skip any `[x]` or `[-]`).
- `[-]` means **deferred** — the item was consciously moved to the monthly plan or horizon items. Do not surface it in daily briefings. It lives in the Monthly Pulse → Horizon Items table until its target month.
- These appear in Action Items without a prefix — they are standing weekly commitments visible every day until done.

### 3b. Blockers
Read `01_Execution/YYYY-[W]WW/Blockers.md` if it exists. Extract:
- `## Waiting On` — all `[ ]` items (cofounder deliverables the vault owner is waiting on)
- `## Meetings` — all meeting entries, with **date-aware classification**:
  - Parse each meeting entry's date from the bold text (e.g. `**Tue Mar 3, 8 PM**`)
  - **Skip** `[x]` (completed) and `[-]` (cancelled) entries entirely — do not surface them
  - Classify remaining `[ ]` entries by date relative to today:
    - **Past** (date < today): flag in `### Flags` as `⚠️ Unresolved meeting (date passed) — mark [-] if cancelled, or update date if rescheduled`
    - **Today**: flag normally with full agenda + `/meeting` reminder (existing behavior)
    - **Tomorrow**: flag + auto-trigger `/meeting-prep` (existing behavior, see Step 3c)
    - **Future** (date > tomorrow, same week): mention in `### Flags` as upcoming — lower priority heads-up, no action needed

These do NOT go into `### Action Items`. They are surfaced in `### Flags` only (see below).

### 3c. Meeting Plans
For each **tomorrow's meeting** detected in Blockers `## Meetings`, check if a matching meeting plan already exists at `02_Projects/[Project]/Meeting Plan/*.md` (match by date in filename). If no plan exists, auto-run `/meeting-prep <project> <date>` to generate one. Then read all meeting plan files matching today's or tomorrow's date and incorporate the agenda items into `### Flags` alongside the Blockers meeting entries — this ensures meeting prep work is visible in the briefing.

### 3d. Deadline Plans
Glob for all `02_Projects/**/Deadline Plan.md` files (recursive — catches nested paths like `Project/Subfolder/`). For each one found, read it and extract:
- All deadlines with status 🟡 or 🔴, or with date within 14 days
- Task queue health (from Step 0b's queue health check)

**Flags:** Surface deadline warnings in `### Flags`:
  `- **Deadline: <project> — <name>** (<days> days, <hours_remaining>h left) — <status>`
  If a task queue is running low (fewer pending hours than next 2 weeks of targets):
  `→ Task queue low — add more tasks to Deadline Plan`

**Today's deadline allocation:**
1. Read all deadline plans within 14 days. For each, note:
   - `weekly_target` (hours this week from Weekly Allocation table)
   - `hours_done_this_week` (sum of Progress Log entries dated this week)
   - `remaining_this_week` = weekly_target − hours_done_this_week
2. Count remaining weekdays this week INCLUDING today.
3. Compute today's share: `remaining_this_week / remaining_weekdays` (round to nearest 0.5h).
4. Pick the next `[ ]` task from each deadline's Task Queue.

**Action Items:** Show: `[Project/Subject] <next queue task> — Xh left this week`. In `## Schedule`, allocate a single time block for deadline work based on the daily share. No partial-task fractions — just schedule the task with a reasonable block. The user adjusts the schedule to their preference.

4. **Gmail** — Use the claude.ai Gmail MCP (`mcp__claude_ai_Gmail__gmail_search_messages`) to search for **all emails received today** (query: `newer_than:1d`, NOT `is:unread`). Reading an email ≠ task done. Summarize actionable items and note total count (N unread / M total).

**Note:** Do NOT query Outlook MCP for calendar or email. Outlook calendar is handled via ICS plugin in Day Planner.

## Action Item States

Action items use three checkbox states:
- `[x]` — **Completed today**
- `[>]` — **Move to tomorrow** (will be auto-carried forward in next day's briefing)
- `[>]` with a date suffix — **Delayed to a specific date** (e.g. `- [>] Draft proposal → 2026-03-30`). Carried forward daily as `[>]` until that date arrives, then promoted to a regular `[ ]` item.

## Output Format

Resolve today's daily note path: compute it as `01_Execution/YYYY-WXX/YYYY-MM-DD.md` (using the current ISO week folder). Do NOT rely on `obsidian daily:path` for the full path — it returns only the filename, not the execution folder path. Use the computed path for all subsequent reads, writes, and appends.

Write the briefing directly into the `## Briefing` section of today's daily note. Use this exact structure:

### Action Items

**Deduplication (critical):** Before writing, merge all items from every source (Weekly Todo, carried, planned, manual, email/vault-extracted) into one list. Two items are duplicates if they refer to the same underlying task — match by **semantic meaning**, not exact text. This includes:
- Same task in different languages (e.g. "课程作业" and "Course assignment")
- Same task with different prefixes (e.g. "carried: Research UK micro-VC list" and Weekly Todo "Research UK micro-VC list")
- Same task at different granularity (e.g. "Scope NLP assignment" and "NLP assignment — report + submit")
- Same task described as different steps of the same goal (e.g. "call dentist" and "Schedule dentist appointment" — calling IS the act of scheduling; "review pitch deck" and "finalize pitch deck" — both refer to working on the deck). When a manual item is a concrete action that fulfills a vaguer Weekly Todo item, merge them and keep the manual version (it's the user's stated intent for today).

When merging duplicates, keep the **most specific/updated version** — manual items take priority over Weekly Todo items (the user explicitly asked for them), carried items over planned items (they have fresher context). Apply the most relevant prefix. Never list the same task twice.

**Daily scoping for deadline items:** Deadline-tagged items from Weekly Todo should be replaced with the next queue task + hours remaining this week (computed in Step 3d). Example: `[Project/Subject] Lecture notes first pass — 3.0h left this week`.

After deduplication and daily scoping, list items in this order:
- Weekly Todo `[ ]` items — standing weekly commitments (no prefix, always shown until done). Deadline items use today's scoped version.
- "carried:" items — Yesterday's [>] items whose delay date is today or had no delay date (only if not already covered by a Weekly Todo item above)
- "deferred:" items — Yesterday's [>] items with a future delay date. Written as `- [>] deferred: <task> → YYYY-MM-DD`. These stay `[>]` (not `[ ]`) so they pass through without cluttering the actionable list. They will be carried forward again tomorrow until the date arrives.
- "planned:" items — From this week's ### This week and ### Tomorrow
- "manual:" items — From command arguments
- New tasks extracted from email, calendar, and vault (checklist format)

### Email Highlights
- **Gmail** — today's emails (N unread / M total) + key actionable items

### Calendar
- See Day Planner sidebar for today's schedule

### Flags
- Outstanding items, decisions needed, risks surfaced from vault state
- **Waiting-on items** (always shown if any `[ ]` items in Blockers `## Waiting On`): Add a "Waiting on" sub-block. Format: `- **Waiting on Alice:** Finalize deliverable by Tue Mar 3`. These are informational — they do NOT appear in `### Action Items` or `## Schedule`.
- **Tomorrow's meetings** (day before): If a Blockers `## Meetings` entry matches **tomorrow's** date, surface the meeting + its auto-generated meeting plan. Format: `- **Meeting tomorrow 8 PM:** ProjectAlpha weekly — see [[Meeting Plan filename]] for agenda`.
- **Today's meetings** (on meeting day): If a Blockers `## Meetings` entry matches **today's** date, surface the meeting + full agenda. Format: `- **Meeting today 8 PM:** ProjectAlpha weekly — review deliverables, align strategy`. Include all sub-bullet agenda items. Add a reminder: `→ after meeting, run /meeting <transcript> to process notes`.

Keep it tight. No padding. Scannable.

## Schedule Population

**Daily budget:** Use the budget from the first argument (or the user's answer to the prompt). Fixed events (meetings, calls) consume from this budget. Remaining budget is split between deadline work and non-deadline tasks. If total action items exceed the budget, not everything gets scheduled — overflow items stay in Action Items but are excluded from the time-blocked schedule.

After writing the briefing, populate the `## Schedule` section of today's daily note with time-blocked items using Day Planner syntax:

```
## Schedule

- [ ] 10:30 - 11:00 Investor call (Alice, Bob, Carol)
- [ ] 11:00 - 12:30 Polish BP — narrative + financials
- [ ] 13:30 - 15:00 NLP assignment — scope + outline
```

Rules:
- **Syntax**: Every schedule item MUST use Day Planner checkbox format: `- [ ] HH:MM - HH:MM Description` (24h clock).
- **Start time**: Use the current time (rounded to the next :00 or :30) as the earliest schedule slot. Do not schedule items before now.
- **Fixed events**: Extract any action item, email, or calendar reference that mentions a specific time → place at that time. These also appear in `### Action Items` for tracking.
- **Time-block today's action items**: Schedule only items scoped to today (per the daily budget). Deadline tasks use the daily allocation from Step 3d. Non-deadline tasks: estimate 30min–1.5h each. If total action items exceed budget, prioritize: (1) fixed-time events, (2) items with tomorrow deadlines, (3) deadline revision work, (4) project work, (5) everything else. Items that don't fit → leave in Action Items unchecked but NOT in the schedule. Add a note: "Overflow — consider pushing to tomorrow."
- The user will adjust the proposed schedule — treat it as a draft, not a prescription.
- If there are truly no action items to schedule, write: `- [ ] No items to schedule. Add time-blocks here as needed.`

## Post-Briefing

After the briefing is written, open the daily note in Obsidian: `obsidian open path="<daily-note-path>"` (using the path resolved from `obsidian daily:path`).

Then automatically run `/daily-github` to fetch today's trending repos and append the summary to the daily note. Pass no arguments (defaults: all languages, daily, 10 repos).