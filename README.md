# Operator

An AI-native personal operating system built on Obsidian + Claude Code.

Operator is an opinionated system of 15 Claude Code skills that turn an Obsidian vault into a structured execution engine — daily briefings, weekly reviews, strategic planning, meeting processing, deadline tracking, and knowledge synthesis, all orchestrated by AI agents.

## Quick Start

### 1. Set up the vault

Copy the folder structure into a new Obsidian vault (or adopt it in an existing one):

```bash
git clone https://github.com/yuhanwang14/obsidian-operator.git
cp -r obsidian-operator/vault-template/* /path/to/your/vault/
cp obsidian-operator/CLAUDE.md /path/to/your/vault/
```

### 2. Install skills

Install all skills at once, or pick the ones you need:

```bash
# Install individual skills
npx skills add yuhanwang14/obsidian-operator@daily-init
npx skills add yuhanwang14/obsidian-operator@weekly-init
npx skills add yuhanwang14/obsidian-operator@weekly-review
npx skills add yuhanwang14/obsidian-operator@meeting
npx skills add yuhanwang14/obsidian-operator@project-init
# ... etc

# Or install all 15
for skill in daily-init weekly-init weekly-review daily-github ai-weekly-digest \
  meeting meeting-prep project-init project-sync quarterly-plan annual-vision \
  deadline-plan decision synthesize organize; do
  npx skills add yuhanwang14/obsidian-operator@$skill
done
```

### 3. Start using

```bash
cd /path/to/your/vault
claude

# Your first day:
/project-init MyProject
/daily-init 6
```

## Prerequisites

| Requirement | Required | Notes |
|-------------|----------|-------|
| [Obsidian](https://obsidian.md) | Yes | The vault app |
| [Claude Code](https://claude.ai/code) | Yes | CLI for Claude |
| [Day Planner](https://github.com/ivan-lednev/obsidian-day-planner) plugin | Recommended | Time-blocking in daily notes |
| [Obsidian CLI](https://github.com/Obsidian-TTRPG-Community/obsidian-cli) | Recommended | Skills fall back to file tools if unavailable |
| [ffmpeg](https://ffmpeg.org/) | Optional | For `/meeting` auto-transcription (`brew install ffmpeg`) |
| Gmail MCP | Optional | For email integration in `/daily-init` |
| [Templater](https://github.com/SilentVoid13/Templater) plugin | Optional | For daily note templates |

## Configuration

After installing, edit the **Customization** table in [CLAUDE.md](CLAUDE.md) to match your setup (vault owner name, calendar names, file paths).

### Secrets (`~/.secrets`)

Some skills need API keys. Create a `~/.secrets` file:

```bash
# Required for /meeting auto-transcription (Mode 1)
export GEMINI_API_KEY="your-key-here"    # https://aistudio.google.com/apikey
```

Then copy the transcription script:

```bash
cp skills/meeting/scripts/gemini-transcribe.sh ~/bin/
chmod +x ~/bin/gemini-transcribe.sh
```

If you skip this, `/meeting` still works — just pass it a transcript file or paste text directly (Modes 2 & 3).

### Gmail MCP (optional)

`/daily-init` can pull today's emails into your briefing via Claude Code's built-in Gmail MCP. To enable:

1. Open Claude Code settings and connect your Google account under MCP integrations
2. Grant Gmail read access when prompted

If not configured, `/daily-init` skips the email section silently.

### Apple Calendar & Reminders (macOS only)

`/deadline-plan` and `/quarterly-plan` can create calendar events and reminders via AppleScript. No setup needed beyond macOS — configure the calendar and reminders list names in [CLAUDE.md](CLAUDE.md).

## Vault Structure

```
00_Strategy/            — annual vision, quarterly plans, monthly pulses
01_Execution/           — daily notes, weekly todos, blockers, reviews
02_Projects/            — per-project folders with meeting plans, transcripts, deadlines
03_Thinking/            — reflections, ideas, mental models
04_Knowledge/           — research, meeting knowledge, decision analyses, digests
05_Archive/             — inactive or completed material
```

See [CLAUDE.md](CLAUDE.md) for full conventions, frontmatter spec, checkbox states, and AI agent instructions.

## Skills Reference

### Daily Operations

| Skill | Description |
|-------|-------------|
| `daily-init` | Generate today's briefing — syncs completions, gathers email/calendar/vault data, produces action items + time-blocked schedule |
| `daily-github` | Fetch trending GitHub repos, write full report to knowledge folder, append summary to daily note |

### Weekly Operations

| Skill | Description |
|-------|-------------|
| `weekly-init` | Create week folder + Weekly Todo — carries items from last week, injects deadline tasks, populates Blockers from calendar |
| `weekly-review` | AI synthesis of the week — progress, stalled items, patterns, intention tracking, horizon items, next-week focus |
| `ai-weekly-digest` | Curated AI landscape digest from RSS feeds + web search — papers, big tech, startups, open-source, policy |

### Strategic Planning

| Skill | Description |
|-------|-------------|
| `quarterly-plan` | Three modes: `init` (set quarterly goals), `review` (end-of-quarter synthesis), `pulse` (monthly checkpoint) |
| `annual-vision` | Annual vision setting or year-end retrospective |

### Knowledge & Synthesis

| Skill | Description |
|-------|-------------|
| `meeting` | Process meeting transcripts (auto-transcribe, chunked, or direct) — produces transcript + knowledge note, routes actions |
| `meeting-prep` | Generate meeting agenda from project context — reads project note, blockers, weekly progress, deadlines |
| `project-init` | Scaffold a new project — creates folder structure + project note with frontmatter |
| `project-sync` | Aggregate knowledge notes + weekly reviews into the project note — Knowledge Base, Weekly Progress, Strategic Signals |
| `deadline-plan` | Backward-schedule deadlines with ramp algorithm — task queues, automatic progress tracking, weekly allocation |
| `decision` | Stress-test a decision — assumptions, risks, alternatives, recommendation |
| `synthesize` | Distill notes into structured knowledge (concept / course / brainstorm templates) |

### Vault Maintenance

| Skill | Description |
|-------|-------------|
| `organize` | Scan vault for misplaced files, missing frontmatter, notes to split/merge (read-only recommendations) |

## System Architecture

### How skills work together

```
                        ┌─────────────────────────────────────────────┐
                        │      NEW-WEEK BOUNDARY /daily-init          │
                        │  (first /daily-init of a new ISO week)      │
                        │                                             │
                        │  1.  /weekly-review  (close last week)      │
                        │  1b. /ai-weekly-digest (AI landscape)       │
                        │  1c. /quarterly-plan pulse (new month)      │
                        │  1d. /quarterly-plan review (new quarter)   │
                        │  1e. /quarterly-plan init  (new quarter)    │
                        │  2.  /weekly-init    (open new week)        │
                        │  3.  briefing        (today's data)         │
                        │  4.  /daily-github   (trending repos)       │
                        └─────────────────────────────────────────────┘

                        ┌─────────────────────────────────────────────┐
                        │      SAME-WEEK /daily-init                  │
                        │  (subsequent days within same ISO week)     │
                        │                                             │
                        │  0.  sync yesterday's [x] → Weekly Todo    │
                        │      sync [-] cancelled → Blockers         │
                        │  0b. sync [x] → Deadline Plans (task queue) │
                        │  1.  (skip weekly transition)               │
                        │  1c. /quarterly-plan pulse (new month)      │
                        │  2.  /weekly-init if missing                │
                        │  3.  briefing        (today's data)         │
                        │  4.  /daily-github   (trending repos)       │
                        └─────────────────────────────────────────────┘
```

### Data flow

```
Daily notes accumulate in 01_Execution/YYYY-WXX/
    ↓ [x] completions sync back to Weekly Todo + Blockers automatically
    ↓ [-] cancelled meetings sync back to Blockers ## Meetings
    ↓ [x] deadline tasks sync back to Deadline Plan task queue + hours
    ↓ [>] items carry forward to next day's briefing
    ↓
/meeting routes actions after processing transcripts:
    → Vault owner's independent actions → Weekly Todo
    → Cofounder deliverables → Blockers.md ## Waiting On
    → Meeting-dependent work → Blockers.md ## Meetings
    ↓
/daily-init reads Blockers.md + Deadline Plans → surfaces in ### Flags:
    → Waiting-on items (always), today's meetings + agenda + /meeting reminder
    → Tomorrow's meetings: auto-runs /meeting-prep if no plan exists
    → Deadline warnings (🟡/🔴, within 14 days) + task queue health
    ↓
/weekly-review reads all daily notes + Weekly Todo + Blockers + projects
    → writes Weekly Review.md (AI synthesis + suggested next-week todos)
    → detects horizon items (monthly/quarterly deferrals & deadlines)
    ↓
/ai-weekly-digest reads GitHub trending files + RSS + web search
    → writes AI Weekly Digest, appends summary to Weekly Review
    ↓
/weekly-init reads Weekly Review "Todos next week" + uncompleted Weekly Todo
    → carries them into new week's Weekly Todo
    → pulls top deadline tasks from task queues into Weekly Todo
    → carries undelivered Blockers ## Waiting On items
    → populates ## Meetings from Weekly Review + ICS calendar data
    ↓
Cycle repeats
    ↓
/quarterly-plan pulse (monthly) reads weekly reviews + horizon items + projects
    → writes Monthly Pulse, assesses quarterly goals (🟢🟡🔴)
    ↓
/quarterly-plan review (end of quarter) reads plan + pulses + weekly reviews
    → writes Quarterly Review, carries unresolved items forward
    ↓
/quarterly-plan init (start of quarter) reads vision + last review
    → guides goal-setting, writes Quarterly Plan
    ↓
/annual-vision reads quarterly reviews + projects
    → writes annual Vision or Annual Review
```

### Knowledge pipeline

```
/meeting-prep → 02_Projects/[P]/Meeting Plan/            (agenda)
/meeting      → 02_Projects/[P]/Meeting Transcripts/     (raw)
              → 04_Knowledge/[P]/Meeting Knowledge/       (synthesis)
              → Weekly Todo + Blockers.md                 (action routing)

/synthesize   → 04_Knowledge/[subfolder]/                 (concept | course | brainstorm)
/decision     → 04_Knowledge/[P]/Decision Challenges/     (stress-test)
/deadline-plan→ 02_Projects/[path]/Deadline Plan.md       (ramp schedule + task queue)
/daily-github → 04_Knowledge/GitHub/                      (daily trending)
/ai-weekly-digest → 04_Knowledge/AI-Weekly/               (weekly AI landscape)
/quarterly-plan   → 00_Strategy/YYYY-QX/                  (plan | review | pulse)
/annual-vision    → 00_Strategy/                          (vision | annual review)

/project-init → 02_Projects/[P]/ + 04_Knowledge/[P]/     (scaffolding)
/project-sync → 02_Projects/[P]/[P].md                   (Knowledge Base + Strategic Signals)
```

### Dependency graph

```
/daily-init ──► /weekly-review (new-week boundary)
            ──► /ai-weekly-digest (new-week boundary)
            ──► /quarterly-plan pulse (new-month boundary)
            ──► /quarterly-plan review (new-quarter boundary)
            ──► /quarterly-plan init (new-quarter boundary)
            ──► /weekly-init (if missing)
            ──► /daily-github (post-briefing)
            ──► /meeting-prep (tomorrow's meetings)

/meeting ───► (self-contained: transcript + knowledge + action routing)

/weekly-review (standalone)
/project-sync  (standalone — pure synthesis)
```

## Customization

This is an opinionated system — the vault structure, note conventions, and skill behaviors are designed to work together. To customize:

1. **CLAUDE.md** is the configuration layer. Edit folder paths, frontmatter fields, checkbox states, or agent behavior there.
2. **Individual skills** can be modified after installation (they live in `~/.claude/skills/` or `~/.agents/skills/`).
3. **Vault structure** can be extended — add new folders as needed. Avoid renaming the core 6 folders without updating CLAUDE.md and skill references.

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for how to get started.

- [Code of Conduct](CODE_OF_CONDUCT.md)
- [Security Policy](SECURITY.md)

## License

[MIT](LICENSE)
