---
name: meeting
description: "Invoke IMMEDIATELY when user shares meeting output: pasted transcripts, dialogue text, .m4a recordings, transcript files, or says a call/sync just ended and wants processing. Responds to /meeting. Produces Obsidian knowledge notes with decisions, actions, and open questions; routes items to weekly todo and blockers. Supports three modes: auto-transcribe (.m4a), chunked transcript directory, and direct/pasted transcript. Not for scheduling, meeting prep, live note-taking, or past meeting recall."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, meeting, transcript, knowledge, action-routing]
---

## Setup (Mode 1 — Auto-transcribe)

Mode 1 requires transcription scripts. Modes 2 and 3 work without them.

**Gemini API transcription (recommended):**
1. Copy `scripts/gemini-transcribe.sh` (bundled in this skill folder) to the path configured in CLAUDE.md (default: `~/bin/`)
2. Set `GEMINI_API_KEY` in the secrets file configured in CLAUDE.md (default: `~/.secrets`)
3. Requires `ffprobe` (from ffmpeg): `brew install ffmpeg`

---

Store a meeting transcript and produce a knowledge note. Supports three modes:
1. **Auto-transcribe** — transcribe a `.m4a` recording via Gemini API, then process
2. **Chunked transcript processing** — stitch transcripts from a directory into vault notes
3. **Direct transcript** — process a single transcript file or pasted text (original behavior)

## Arguments

- `/meeting /path/to/recording.m4a "Meeting Name" [project]` — Mode 1: auto-transcribe
- `/meeting /path/to/meeting/directory/` — Mode 2: chunked transcript processing
- `/meeting /path/to/transcript.txt` — Mode 3: single file transcript
- `/meeting Speaker 1: Hi... Speaker 2: Thanks...` — Mode 3: pasted transcript

If no argument is provided, ask the user to paste the transcript or provide a file/directory path.

## Mode detection

```
if arg1 ends with .m4a        → Mode 1 (auto-transcribe)
if arg1 is a directory path   → Mode 2 (chunked transcript processing)
if arg1 is a file path (.txt/.md) → Mode 3 (single-file transcript)
if arg1 is raw text           → Mode 3 (pasted transcript)
```

---

## Pre-flight check — Speaker information

Before processing any transcript (all modes), verify that the user has provided **speaker information** — at minimum the name of each participant.

1. If the transcript uses generic labels (`Speaker 1`, `Speaker 2`, `S1/S2`, `Person A`, etc.) and the user has **not** supplied a mapping, **pause and ask**:

   > The transcript uses generic speaker labels. To produce accurate notes, I need real names for each speaker. Please provide a mapping, e.g.:
   >
   > - Speaker 1 → Jason
   > - Speaker 2 → Yuhan
   >
   > (At minimum, tell me who was in the meeting.)

2. If the user provides names, apply the mapping to speaker labels throughout the transcript before continuing.
3. If speaker names are already present in the transcript (e.g. `Jason:`, `Yuhan:`) or the user provided them in the invocation, skip this check and proceed.

Do **not** guess or infer speaker identities from transcript content alone — always confirm with the user.

---

## Mode 1 — Auto-transcribe (`/meeting <m4a_path> <name> [project]`)

Detects that the first argument is an `.m4a` file.

### Step 1 — Gather info

- `name` is the second argument (the meeting name). If missing, ask the user.
- `project` is the optional third argument. If missing, list folders under `02_Projects/` and ask the user to pick one.

### Step 2 — Create meeting directory & move audio

1. Create: the meeting directory under the recordings base configured in CLAUDE.md (default: `~/Work/<Project>/Meetings/<name>/`)
2. Get audio duration via `ffprobe -v error -show_entries format=duration -of csv=p=0 "<m4a_path>"`
3. Move (mv) original file into the directory as `<name>.m4a`

### Step 3 — Transcribe audio

#### 3a. Gemini API (primary)

1. Run: `transcript=$(source <secrets_file> && <transcription_script> "<meeting_dir>/<name>.m4a")` (paths from CLAUDE.md Customization table)
2. If successful, write to `<meeting_dir>/<name>.md` and print: `"Gemini transcription complete."`
3. If fails → fall through to 3b

#### 3b. Manual fallback (last resort)

Only reached if Gemini fails.

1. If duration > 20 minutes, split into 20-minute chunks:
   ```bash
   ffmpeg -i "<meeting_dir>/<name>.m4a" -ss 0 -t 1200 -c copy "<name>_00.m4a" -ss 1200 -t 1200 -c copy "<name>_01.m4a" ...
   ```
   Create blank `.md` for each chunk.
2. Print next steps for manual transcription upload.
3. **Stop here — do not proceed to transcript processing.**

### Step 4 — Auto-continue (Gemini success only)

If transcription succeeded (3a), proceed directly to **Shared Steps** (Step 2 — Extract metadata) using the transcript in `<meeting_dir>/<name>.md`. Do not stop or ask the user.

---

## Mode 2 — Chunked transcript processing (`/meeting <directory_path>`)

Detects that the argument is a directory path.

### Step 1 — Read and stitch transcripts

1. Find all `.txt` and `.md` files in the directory
2. Sort them by filename (so chunk_00, chunk_01, etc. are in order)
3. Read each file and concatenate into a single transcript, inserting `--- Chunk N ---` markers between segments:
   ```
   --- Chunk 1 ---
   [contents of first file]

   --- Chunk 2 ---
   [contents of second file]
   ```

### Step 2 — Infer project

Extract the project name from the directory path. The expected structure matches the recordings base configured in CLAUDE.md (default: `~/Work/<Project>/Meetings/<name>/`), so the project is the folder name after the base directory.

If the path doesn't match this pattern, ask the user which project this belongs to (list folders under `02_Projects/`).

### Step 3 — Continue with Steps 2–5 below

Use the stitched transcript and proceed to **Step 2 — Extract metadata** in the shared steps.

---

## Mode 3 — Direct transcript (existing behavior)

If the argument looks like a file path to a `.txt` or `.md` file (starts with `/` or `~`), read the file. Otherwise treat the argument itself as the transcript text.

Proceed directly to **Step 2 — Extract metadata** in the shared steps.

---

## Shared Steps (used by Mode 2 and Mode 3)

### Step 2 — Extract metadata

From the transcript content and any available context, determine:

- **date** — look for a date mentioned in the transcript or ask the user. Format as `YYYY-MM-DD`.
- **project** — infer from names, topics, or context in the transcript. Cross-reference vault projects in `02_Projects/`. If already determined by Mode 2, use that. If unclear, ask the user.
- **participants** — use speaker labels from the transcript (e.g. `Speaker 1`, `Jason`, etc.). If names are known from vault context, use real names.
- **duration** — if mentioned in the transcript or filename, use it. Otherwise omit.
- **session** — if the filename or transcript implies a session number, include it. Otherwise omit.
- **language** — infer from transcript content (e.g. "Mandarin/English" if mixed).

If date or project cannot be inferred confidently, ask the user before proceeding.

### Step 3 — Create the transcript note

**File naming convention:**
- Short date format: `M.DD.md` (e.g. `2.20.md`)
- For numbered sessions: `M.DD-N.md` (e.g. `2.19-2.md`)
- Save to `02_Projects/[Project]/Meeting Transcripts/`; create the folder if needed.

```
---
type: meeting
date: YYYY-MM-DD
project: [Project]
participants: [Speaker 1, Speaker 2]
duration: "HH:MM:SS"   # omit if unknown
session: N of M        # omit if not applicable
language: [Language]
---

## Summary

One-sentence framing of the meeting, then 3–5 bullets:

- **[Sub-topic]:** [description]

---

## Transcript

**Language:** [Language]

**Setting:** [inferred — e.g. Zoom call, In-person, Phone call]

[paste the transcript here exactly as provided, preserving speaker labels and timestamps]
```

### Step 4 — Create the knowledge note

**File naming convention:**
- Full date format: `YYYY-MM-DD - [Short Topic].md` (topic = 2–4 words)
- For numbered sessions: `YYYY-MM-DD Session N - [Short Topic].md`
- Save to `04_Knowledge/[Project]/Meeting Knowledge/`; create the folder if needed.
- The `source` field uses the transcript's short filename (e.g. `"[[2.20]]"`).

```
---
type: knowledge
kind: meeting
date: YYYY-MM-DD
project: [Project]
session: [N of M]   # omit if not applicable
source: "[[M.DD]]"
tags: []
---

## Context

What prompted this meeting and what the goal was.

## Key Points

Main discussion threads — bulleted with **bold sub-labels**.

## Decisions

Concrete things resolved or committed to. If nothing decided, write "No firm decisions reached."

## Open Questions

Explicitly unresolved or deferred items.

## Actions

- [ ] [Speaker or role]: [action]

## Links

- Source: [[M.DD]]
- Related: [[Project Note]]
```

Always include the main project note (e.g. `[[ProjectName]]`) in Related links.

### Step 5 — Route actions to execution layer

After writing the knowledge note, route actions from `## Actions` directly into the execution layer. This happens immediately — the context is freshest right after processing the transcript.

**CLI fallback:** If any `obsidian` CLI command fails, silently use the equivalent file tool (Read, Write, Edit). Do not surface CLI errors to the user.

1. Read the `## Actions` section from the knowledge note just created.
2. For each action assigned to **the vault owner** (configured in CLAUDE.md) that is actionable this week (no future gate, no "after X" condition, no dependency on unfinished work):
   - Append to Weekly Todo (`01_Execution/YYYY-WXX/Weekly Todo.md`): `- [ ] <action text>`
   - Skip if an identical item already exists (exact text match — not fuzzy)
   - **Skip future-gated items** ("after X", "mid-March", "end of March", "once pitch deck is done") — these stay in the knowledge note and flow through `/weekly-review` → `/weekly-init` naturally.
3. For each action assigned to **someone else** (cofounder or collaborator):
   - Append to Blockers `## Waiting On` (`01_Execution/YYYY-WXX/Blockers.md`): `- [ ] Assignee: <action> — [[source note]]`
   - Create Blockers.md with standard frontmatter and empty `## Waiting On` / `## Meetings` sections if it doesn't exist.
   - Skip if an identical item already exists (exact text match).
4. For **meeting-dependent items** (actions assigned to "All"/"Team", or vault owner actions requiring synchronous collaboration — "review together", "align with team", "mock pitch", etc.):
   - Append to Blockers `## Meetings` as a sub-bullet under the relevant meeting entry.
   - If no meeting entry exists, create one: `- [ ] **Date** — Meeting name — [[source note]]` with the item as a sub-bullet.

### Step 6 — Report and next steps

- Transcript saved to: `[path]`
- Knowledge note saved to: `[path]`
- Actions routed: N items to Weekly Todo, N to Blockers
- One-sentence description of what the meeting was about.

---
**Next steps:**
1. Run `/project-sync $project` to update the project's Knowledge Base and Strategic Signals
