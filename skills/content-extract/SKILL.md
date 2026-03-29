---
name: content-extract
description: "Scan yesterday's vault notes and newsletter emails for publishable content ideas and append them to the content backlog. TRIGGER when the user wants to find content ideas, mine notes for posts, scan for publishable insights, extract content from their vault, or asks 'what should I write about'. Also triggers for /content-extract. Runs automatically as part of /daily-init post-briefing. Evaluates daily notes, knowledge notes, meeting syntheses, decisions, AI digests, thinking notes, and Substack newsletter emails against four content pillars (founder narrative, AI observer, builder workflow, personal reflection). NOT for drafting content (use /content-draft), not for organizing the vault (use /organize), not for synthesizing notes into knowledge (use /synthesize)."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, content, extraction, backlog, publishing]
---

Scan yesterday's notes for publishable insights and append content suggestions to the backlog.

## Arguments

**Optional date override:** `/content-extract 2026-03-25` scans that specific day instead of yesterday.

If no argument is provided, scan yesterday's date.

## Content Pillars

Every suggestion maps to exactly one pillar:

- **Founder narrative** — building in public, startup decisions, lessons from the journey
- **AI observer** — AI landscape insights, paper highlights, industry trend analysis
- **Builder workflow** — tools, systems, automation, how-you-actually-work content
- **Personal reflection** — career thinking, learning journeys, connecting personal to universal

## Step 1 — Determine scan date

If an argument is provided, use that date. Otherwise compute yesterday's date.

Resolve the scan date to:
- Daily note path: `01_Execution/YYYY-WXX/YYYY-MM-DD.md`
- ISO week folder: `01_Execution/YYYY-WXX/`

## Step 2 — Gather candidate notes

Read notes from these sources (skip any that don't exist):

1. **Daily note** — the scan date's daily note. Focus on `## Briefing > ### Action Items`, `## Plan`, and any free-text sections. Skip the schedule and mechanical items.

2. **Knowledge notes** — Glob `04_Knowledge/**/*.md` for files modified on the scan date. This catches:
   - Meeting knowledge (`04_Knowledge/*/Meeting Knowledge/`)
   - Decision challenges (`04_Knowledge/*/Decision Challenges/`)
   - Synthesized concepts (`04_Knowledge/`)
   - Any other knowledge notes created that day

3. **Thinking notes** — Glob `03_Thinking/**/*.md` for files modified on the scan date.

4. **AI Weekly Digest** — If a file matching `04_Knowledge/AI-Weekly/YYYY-WXX - AI Weekly Digest.md` was created during the scan date's week and hasn't been scanned before (not already in backlog), include it.

5. **GitHub Trending** — If `04_Knowledge/GitHub/YYYY-MM-DD - GitHub Trending*.md` exists for the scan date, include it.

6. **Newsletter emails** — Query Gmail for Substack newsletters received on the scan date using `mcp__claude_ai_Gmail__gmail_search_messages` with query `from:substack.com after:YYYY/MM/DD before:YYYY/MM/(DD+1)`. For each newsletter email, read the full message body with `mcp__claude_ai_Gmail__gmail_read_message`. Focus on the essay content — skip promotional footers, subscription CTAs, and "like/comment/share" boilerplate. Treat each newsletter as a candidate note for Step 3 evaluation. When appending to the backlog, use the newsletter title as the `[[source]]` wiki-link text (no actual vault note exists — this is fine) and set `from:newsletter`.

7. **Catch-up pass (unscanned this week)** — After the date-specific scan above, also glob `04_Knowledge/**/*.md` and `03_Thinking/**/*.md` for files modified this week (same ISO week as the scan date) whose `[[wiki-link]]` does NOT already appear in `06_Content/Backlog.md`. This catches meeting notes, decisions, and essays from earlier in the week that were never scanned — for example, a meeting note from Monday that didn't trigger a scan until Thursday's daily-init. Only include notes that pass the Step 3 evaluation criteria.

Read each candidate note. If a note is very long (>200 lines), read only the first 100 lines and any summary/synthesis sections.

## Step 3 — Evaluate for publishable insights

For each candidate note, ask: **Is there a non-obvious insight here that would be valuable to an external audience?**

Criteria for inclusion:
- Contains an opinion, observation, lesson, or counterintuitive finding — not just facts
- Has enough substance for at least a LinkedIn post (~200 words of material)
- Maps clearly to one of the four pillars
- Would be interesting to someone who doesn't have vault context

Exclude:
- Pure logistics (meeting scheduling, todo status updates, calendar items)
- Raw data without interpretation (repo lists without commentary, email summaries)
- Private/personal items (health, finances, relationship notes)
- Notes that are purely internal project status without a generalizable lesson

Cap at **3 suggestions per scan**. If more than 3 candidates qualify, pick the strongest — prioritize notes with clear opinions or lessons over raw information.

If nothing qualifies, that's fine. Output nothing. Not every day produces publishable material.

## Step 4 — Append to backlog

Read `06_Content/Backlog.md`. For each suggestion:

**Deduplication check:** If the source note's `[[wiki-link]]` already appears anywhere in the backlog (Queue or Parked), skip it.

**Append format** — add under `## Queue`:
```
- [ ] **<pillar>** · <one-line hook, max 15 words> · [[source note filename]] · `from:<origin>`
```

Where:
- `<pillar>` is one of: Founder narrative, AI observer, Builder workflow, Personal reflection
- `<one-line hook>` is a punchy, specific description of the content angle (not a generic topic)
- `[[source note filename]]` is an Obsidian wiki-link to the source
- `<origin>` is the skill/source that created the note: `daily`, `meeting`, `synthesize`, `decision`, `ai-weekly-digest`, `daily-github`, `thinking`

**Hook writing guidance:** The hook should make someone want to read more. It's the content angle, not the topic.

Bad: "AI developments this week" (too vague)
Good: "Why Claude's 1M context window changes retrieval architectures"

Bad: "Meeting with investors" (topic, not angle)
Good: "The question every VC asked that we couldn't answer"

## Step 5 — Report (daily-init integration)

When running as part of `/daily-init`, append to today's `### Flags` section:

```
- **Content:** N new ideas added to backlog (pillar1, pillar2)
```

If zero ideas were added, output nothing — no noise in the briefing.

When running standalone, print a summary to the user: how many ideas found, the hooks, and which pillars they map to.
