---
name: content-draft
description: "Generate platform-specific content drafts from vault notes, backlog items, or free topics. TRIGGER when the user wants to draft a LinkedIn post, write a tweet thread, create a blog article, write a newsletter, or turn a note into publishable content. Signal phrases: 'draft a post about', 'turn this into a LinkedIn post', 'write a thread about', 'create content from', 'draft from this note', 'write up this idea'. Also triggers for /content-draft. Supports multiple output formats: LinkedIn (delegates to linkedin-content skill), Twitter/X threads, non-technical articles, technical blogs (delegates to technical-blog-writing skill), and newsletters. NOT for extracting content ideas (use /content-extract), not for synthesizing notes (use /synthesize), not for meeting processing (use /meeting)."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, content, drafting, linkedin, twitter, blog, newsletter]
---

Generate platform-specific content drafts from backlog items, vault notes, or free topics.

## Arguments

Three input modes:

1. **No args** — `/content-draft` — presents the backlog for selection
2. **Note path** — `/content-draft 04_Knowledge/AI-Weekly/2026-W12 - AI Weekly Digest.md` — drafts from that note
3. **Free topic** — `/content-draft "Why generation is not creation"` — drafts from a topic string

## Step 1 — Resolve source material

### Mode 1: From backlog (no args)
Read `06_Content/Backlog.md`. Present all `[ ]` items from `## Queue` to the user as a numbered list. Ask which item to draft. Read the selected item's `[[source note]]` link.

### Mode 2: Direct note (path arg)
Read the specified note directly.

### Mode 3: Free topic (quoted string)
Search the vault (Grep across `03_Thinking/`, `04_Knowledge/`, `01_Execution/`) for notes related to the topic. If matches found, read the top 2-3 most relevant. If no matches, proceed with the topic string alone — the draft will be generated from the user's knowledge and the topic.

### Expand context
After reading the primary source, look for linked context:
- If the source mentions a project → read the project note (`02_Projects/[Project]/[Project].md`)
- If the source is a meeting note → check for related decision notes or knowledge notes
- If the source has `[[wiki-links]]` → read up to 3 linked notes for context

Keep total context under ~2000 lines. Prioritize depth on the primary source over breadth.

## Step 2 — Identify pillar

If the source came from a backlog item, use its pillar tag. Otherwise, infer:

| Signal | Pillar |
|--------|--------|
| Startup decisions, fundraising, team, product pivots | **Founder narrative** |
| AI papers, model releases, industry trends, research | **AI observer** |
| Tools, automation, workflows, systems thinking | **Builder workflow** |
| Career, learning, identity, philosophy | **Personal reflection** |

If unclear, ask the user.

## Step 3 — Select output format(s)

Ask the user which format(s) to generate:

| Format | How it's generated |
|--------|-------------------|
| **LinkedIn post** | Delegates to `linkedin-content` skill |
| **Twitter/X thread** | Built-in (see Thread Format below) |
| **Non-technical article** | Built-in, uses Voice Guide (see Article Format below) |
| **Technical blog** | Delegates to `technical-blog-writing` skill |
| **Newsletter edition** | Built-in (see Newsletter Format below) |
| **All** | Generates all applicable formats |

## Step 4 — Generate drafts

### LinkedIn Post
Invoke the `linkedin-content` skill. Pass it:
- The assembled source material and context
- The pillar and content angle
- Let the skill handle hook formula, formatting, hashtags, and CTA

### Twitter/X Thread Format
Generate a thread of 3-7 tweets:

```
🧵 1/ [Hook — the provocative claim or question that stops the scroll]

2/ [Setup — the context or conventional wisdom you're challenging]

3/ [Core insight — the non-obvious thing you learned/observed]

4/ [Evidence or example — concrete, specific, from your experience]

5/ [Implication — "here's what this means for..."]

6/ [Open question or CTA — invite engagement, don't close the loop]
```

Rules:
- Each tweet ≤280 characters
- Tweet 1 is everything — it must create tension or curiosity
- No hashtags in the thread body (optional in final tweet only)
- Use line breaks for readability within tweets
- Concrete > abstract. "We lost 40% of signups" > "we had retention issues"

### Non-Technical Article Format
Read `06_Content/Voice Guide.md` and load the `## Non-Technical Article` profile.

If the profile exists, follow its patterns. If empty/TBD, use this default structure:

```
# [Title — specific, not clickbait]

[Opening: concrete external event or personal moment]

[Pivot: the deeper question this raises]

---

[Argument section 1: the conventional view and why it's incomplete]

[Argument section 2: the insight, grounded in experience or evidence]

---

[Honest self-audit: where your argument might be wrong]

[Closing: an unresolved question, not a neat conclusion]
```

Target: 1000-2500 words. Use bold for key distinctions, blockquotes for pivotal questions.

After generating, suggest the user add the final published version to `06_Content/Voice Guide.md` to calibrate future drafts.

### Technical Blog
Invoke the `technical-blog-writing` skill. Pass it:
- The source material and technical context
- The target audience level
- Let the skill handle structure, code examples, and formatting

### Newsletter Format
```
# [Newsletter title — issue number or date]

> [1-2 sentence hook — why this edition matters]

## [Section 1 title]
[Key insight, 150-300 words]

## [Section 2 title]
[Key insight, 150-300 words]

## [Section 3 title]
[Key insight, 150-300 words]

---

**What I'm thinking about:** [Personal aside — 2-3 sentences connecting themes]

**Worth reading:** [1-2 external links with one-line commentary]
```

Target: 800-1500 words total. Designed for email readability — short paragraphs, clear headers, scannable.

## Step 5 — Write outputs

Create the draft folder and write files:

```
06_Content/Drafts/YYYY-MM-DD - <slug>/
  linkedin.md      (if LinkedIn was selected)
  twitter.md       (if Twitter was selected)
  article.md       (if article or technical blog was selected)
  newsletter.md    (if newsletter was selected)
```

The `<slug>` is a URL-friendly version of the content hook (lowercase, hyphens, max 50 chars).

Each draft file includes frontmatter:
```yaml
---
pillar: <pillar name>
source: <source note path>
format: <linkedin|twitter|article|newsletter>
status: draft
date: YYYY-MM-DD
---
```

## Step 6 — Update backlog

If the source was a backlog item, update it in `06_Content/Backlog.md`:
- Append ` · → [[YYYY-MM-DD - <slug>/]]` to the item line

Do not mark it `[x]` — that happens when the user actually publishes.

## Step 7 — Present to user

Show the user what was generated:
- List the files created with their paths
- For short formats (LinkedIn, Twitter), show the full draft inline
- For long formats (article, newsletter), show the first ~500 chars and the file path
- Remind: "Edit these drafts to your satisfaction, then publish. When published, mark the backlog item `[x]` and move to Published/."
