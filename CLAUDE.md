# CLAUDE.md

> **Setup:** Copy this file into your Obsidian vault root to configure Claude Code for the Operator system.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an **Obsidian vault** named "Operator" — a personal operating system for research, startups, learning, and long-term thinking. It stores projects, execution plans, reflections, and structured knowledge, and is designed to work with AI agents for synthesis, analysis, and decision support.

It is not a software development project; there are no build systems, test runners, or package managers.

## Folder Structure

```
00_Strategy/            — annual vision, quarterly plans, monthly pulses
  YYYY Vision.md        — annual north star + goals (created by /annual-vision)
  YYYY-QX/              — quarter subfolder (e.g. 2026-Q1/)
    Quarterly Plan.md   — objectives + milestones (created by /quarterly-plan init)
    Quarterly Review.md — end-of-quarter synthesis (created by /quarterly-plan review)
    Monthly Pulse - MM.md — monthly checkpoint (created by /quarterly-plan pulse)
01_Execution/           — active plans, sprints, decisions, daily/weekly ops
  YYYY-WXX/             — week subfolder (e.g. 2026-W08/)
    YYYY-MM-DD.md       — daily notes (created by Templater, moved here automatically)
    Weekly Todo.md      — flat checklist for the week (created by /weekly-init)
    Blockers.md         — cofounder deliverables + meeting agendas (created by /weekly-init, populated by /meeting)
    Weekly Review.md    — AI synthesis of the week (created by /weekly-review)
02_Projects/            — per-project folders (e.g. ProjectAlpha, ProjectBeta)
  [Project]/
    [Project].md        — main project note (created by /project-init, synced by /project-sync)
    Deadline Plan.md    — ramp-scheduled deadline tracker (created by /deadline-plan, nested paths supported)
    Meeting Plan/       — meeting agendas (created by /meeting-prep, auto-triggered by /daily-init)
    Meeting Transcripts/— raw transcripts (created by /meeting)
03_Thinking/            — reflections, ideas, mental models, long-form reasoning
04_Knowledge/           — research, references, synthesized knowledge
  [Project]/            — project-specific knowledge
    Meeting Knowledge/  — meeting synthesis notes (created by /meeting)
    Decision Challenges/— decision stress-tests (created by /decision)
  GitHub/               — daily trending repo reports (created by /daily-github)
  AI-Weekly/            — weekly AI landscape digests (created by /ai-weekly-digest)
05_Archive/             — inactive or completed material
06_Content/             — content backlog, drafts, and published archive
  Backlog.md            — master content idea queue with pillar tags (created by /content-extract)
  Voice Guide.md        — per-format voice profiles for /content-draft
  Drafts/               — platform-specific draft folders (created by /content-draft)
  Published/            — archived published items
```

Do not reorganize files across folders unless explicitly asked.

## Note Conventions

**Frontmatter** is used only for project and execution notes (not all notes). Common fields:

```yaml
---
type: project | meeting | research | reflection | idea
status: active | paused | archived
date: YYYY-MM-DD
project: <name>
---
```

**Standard headings** used across most notes:

```
## Focus
## Context
## Insights
## Tasks
## Open Questions
```

**Checkbox states** in daily notes:
- `[ ]` — todo
- `[x]` — completed
- `[>]` — carry forward to tomorrow (auto-picked up by next day's `/daily-init`)
- `[-]` — skipped/cancelled (deferred items in Weekly Todo; cancelled meetings in Blockers)

**Blockers.md sections** (per-week dependency tracking):
- `## Waiting On` — cofounder deliverables with `Assignee:` prefix + source wiki-link. Mark `[x]` when delivered.
- `## Meetings` — upcoming meetings with sub-bullet agendas for meeting-dependent work. Mark `[x]` when meeting happens.

Notes use Obsidian wiki-link syntax: `[[Note Title]]`. Avoid adding frontmatter or restructuring headings in notes that don't already use them.

## AI Agent Role

When working in this vault, operate at a **strategic and synthesis level**:

- Distill raw notes into structured insights
- Generate and refine roadmaps and execution plans
- Review major decisions and surface risks
- Synthesize research and market information
- Identify patterns, bottlenecks, and leverage points
- Propose next actions aligned with long-term strategy

Do not micromanage trivial tasks unless explicitly requested. Prefer high-leverage observations and recommendations over exhaustive lists.

## Enabled Core Plugins

Active and relevant: Daily Notes, Templates, Canvas, Bases, Backlinks, Outgoing Links, Graph, Bookmarks, Outline, Tags, Properties, File Recovery, Sync.

Disabled: slash-command, footnotes, zk-prefixer, workspaces, publish, audio-recorder, slides, webviewer.

Avoid modifying `.obsidian/` configuration files unless specifically asked.

## Customization

Update these values to match your setup. Skills reference them as "the vault owner", "the configured calendar name", etc.

| Setting | Value | Used by |
|---------|-------|---------|
| Vault owner name | `Yuhan` | `/meeting`, `/daily-init` |
| Apple Calendar name | `Operator` | `/deadline-plan`, `/quarterly-plan` |
| Apple Reminders list | `Operator` | `/deadline-plan`, `/quarterly-plan` |
| Meeting recordings base | `~/Work/<Project>/Meetings/` | `/meeting` |
| Transcription script | `~/bin/gemini-transcribe.sh` | `/meeting` |
| Secrets file | `~/.secrets` | `/meeting` |

## Obsidian CLI

The Obsidian CLI (`obsidian`) is available in PATH and requires the Obsidian app to be running. It routes changes through Obsidian's API so open files update immediately — no filesystem event delay.

**Prefer CLI for:**
- Task toggling: `obsidian task done path="01_Execution/2026-W09/Weekly Todo.md" line=N`
- Appending to open files: `obsidian append path="..." content="..."`
- Opening files in Obsidian: `obsidian open path="01_Execution/2026-W09/2026-02-28.md"`
- Vault search: `obsidian search:context query="MyProject" path="01_Execution"`
- Daily note operations: `obsidian daily:path`, `obsidian daily:read`, `obsidian daily:append content="..."`

**Keep file tools for:**
- Full-file rewrites (Write tool) — Weekly Review, project notes, knowledge notes
- Heading-targeted inserts (Edit tool) — briefing into daily note, digest into Weekly Review
- Creating new structured notes (Write tool)
- Reading files during synthesis (Read tool)

**Fallback rule:** If CLI fails (e.g. Obsidian not running), silently fall back to the equivalent file tool. Do not surface the error to the user.

**Path convention:** Always use `path=` (exact relative path from vault root), not `file=` (name-based lookup).
