---
name: decision
description: "TRIGGER when the user wants to stress-test, challenge, analyze, or evaluate a decision — whether described directly, referenced as a file, or pasted as a proposal. Signal phrases: 'stress-test this decision', 'challenge this proposal', 'should we pivot', 'is this the right call', 'what are the risks of choosing X', 'analyze the tradeoffs', 'decision analysis'. Also triggers for /decision. Produces a structured decision challenge note with assumptions, risks, alternatives, and recommendation. NOT for making decisions on the user's behalf, brainstorming new ideas, general project status, or meeting prep."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, decision, analysis, risk, strategy]
---

Stress-test the decision described in my arguments.

## Arguments

This command accepts either:
- A **file path** or `@file` reference to a note containing the decision/proposal to challenge
- A **description** of the decision pasted directly as the argument

Example: `/decision @"02_Projects/MyProject/Architecture.md"`
Example: `/decision Should we pivot from B2C to B2B for MyProject?`

If no argument is provided, ask the user what decision to stress-test.

## Step 1 — Read the source

If the argument is a file path or `@file` reference, read the file. Otherwise treat the argument itself as the decision description.

Determine:
- **project** — infer from file location, content, or vault context. If unclear, ask.
- **source** — the file or topic being challenged.
- **decision topic** — a 2–5 word summary of what's being decided.

## Step 2 — Analyze

Work through each section. Be direct. Surface uncomfortable things.

1. **Decision** — restate what's being decided clearly in 1–2 sentences
2. **Assumptions** — what needs to be true for this to be the right call
3. **Risks** — what could go wrong, what's being overlooked
4. **Alternatives** — what other options exist and why they were (or should be) ruled out
5. **Recommendation** — given the above, what's the strongest path forward

## Step 3 — Save the note

Use this format:

```
---
type: knowledge
kind: decision
source: "[source file or description]"
date: [today YYYY-MM-DD]
project: [Project]
tags: [relevant tags]
---

# [Decision Topic] — Decision Challenge

## Decision

[Restated clearly]

## Assumptions

[Bulleted list — what must be true]

## Risks

[Bulleted list — what could go wrong, what's overlooked]

## Alternatives

[Lettered options with analysis of each]

## Recommendation

[Direct recommendation with reasoning]

## Links

- Source: [[source note if applicable]]
- Related: [[Project Note]]
```

**File naming:** `YYYY-MM-DD - [Decision Topic].md`
**Save to:** `04_Knowledge/[Project]/Decision Challenges/`; create the folder if needed.

If the decision isn't tied to a project, save to `04_Knowledge/Decisions/`.

## Step 4 — Report back

- Decision challenge saved to: `[path]`
- One-sentence summary of the recommendation.

## Step 5 — Next steps

---
**Next steps:**
1. If this decision affects project direction, update `## Now` or `## Risks` in the project note
2. Run `/project-sync <project>` to include this analysis in the project's Knowledge Base
