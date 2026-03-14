---
name: ai-weekly-digest
description: "Generate a curated weekly AI landscape digest covering top papers, big tech announcements, early-stage startup funding, open-source releases, and strategic context. TRIGGER when the user asks for an AI digest, AI weekly roundup, what happened in AI this week, or wants to catch up on AI news and research. Also triggers for /ai-weekly-digest. Pulls from RSS feeds, web search, and vault GitHub trending data. NOT for general news, non-AI topics, daily briefings, or project-specific research."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, ai, research, digest, weekly]
---

Curated weekly AI landscape digest: top papers, big tech moves, startup funding, open-source releases, and policy developments.

## Arguments

This command accepts an optional argument: a target week.

- **`last`** — target last week (e.g. `/ai-weekly-digest last`)
- **`YYYY-WXX`** — target a specific ISO week (e.g. `/ai-weekly-digest 2026-W08`)
- **No argument** — auto-detect: if today is Monday, target last week; otherwise target the current week.

Examples:
- `/ai-weekly-digest` — auto (last week on Monday, current week otherwise)
- `/ai-weekly-digest last` — explicitly target last week
- `/ai-weekly-digest 2026-W08` — target a specific week

## Step 1 — Determine scope & guard

- **Target week:** Parse the optional argument:
  - If `last` → target last week.
  - If `YYYY-WXX` → target that specific week.
  - If no argument → Monday targets last week, other days target the current week.
- Compute the week identifier as `YYYY-WXX` (ISO week) and its date range (Monday–Sunday).
- **Guard:** If `04_Knowledge/AI-Weekly/YYYY-WXX - AI Weekly Digest.md` already exists, print "AI Weekly Digest for YYYY-WXX already exists — skipping." and stop.

## Step 2 — Gather open-source data from vault

Read all `04_Knowledge/GitHub/YYYY-MM-DD - GitHub Trending*.md` files whose dates fall within the target week (Monday–Sunday).

- Extract all repos across those files: name, description, stars gained, relevance notes.
- Deduplicate by repo name across days; sum stars gained for repos appearing multiple days.
- Rank by cumulative stars gained.
- Collect all `## Patterns & Themes` sections for synthesis in Step 6.

If no GitHub trending files exist for the target week, note this and proceed — the Open-Source section will rely on WebSearch.

## Step 3 — Fetch RSS feeds

Fetch all of the following feeds **in parallel** using WebFetch:

| # | Feed URL | Section |
|---|----------|---------|
| 1 | `https://arxiv.org/rss/cs.AI` | Academic |
| 2 | `https://arxiv.org/rss/cs.CL` | Academic (NLP/LLM) |
| 3 | `https://arxiv.org/rss/cs.LG` | Academic (ML) |
| 4 | `https://blog.openai.com/rss/` | Big Tech |
| 5 | `https://blog.google/technology/ai/rss/` | Big Tech |
| 6 | `https://ai.meta.com/blog/rss/` | Big Tech |
| 7 | `https://www.anthropic.com/rss.xml` | Big Tech |
| 8 | `https://blogs.microsoft.com/ai/feed/` | Big Tech |
| 9 | `https://machinelearning.apple.com/rss.xml` | Big Tech |
| 10 | `https://techcrunch.com/category/artificial-intelligence/feed/` | Startups |
| 11 | `https://the-decoder.com/feed/` | Cross-cutting |
| 12 | `https://www.technologyreview.com/feed/` | Policy |

- Filter entries to the target week's date range (Monday–Sunday).
- If a feed fails, log a warning and continue with the rest.

## Step 4 — Filter, rank, and fill gaps with WebSearch

### Academic (3–5 papers)
- Cross-reference arXiv entries with WebSearch for community buzz (e.g. `"best AI papers this week"` on Reddit, X, Hacker News).
- For each selected paper, run one WebSearch for context on its significance.
- Select 3–5 papers that are most impactful or widely discussed.

### Big Tech (5–8 items)
- Supplement blog feed entries with WebSearch for major announcements not covered by blogs (e.g. `"OpenAI announcement this week"`, `"Google AI launch this week"`).
- Select 5–8 most significant items.

### Early-Stage Startups (5–8 items)
- Focus on **pre-seed, seed, and Series A** rounds — not mega-rounds from established players.
- Supplement TechCrunch and The Decoder entries with WebSearch for early-stage AI startup funding, accelerator cohorts, and new product launches (e.g. `"AI startup seed round this week"`, `"AI pre-seed funding this week"`, `"YC AI startups"`, `"AI accelerator cohort 2026"`).
- Prioritize startups relevant to vault projects (personalization, developer tools, local-first AI, agent infrastructure).
- For each item: company name, stage, raise amount, what they're building, and why it matters competitively or as inspiration.

### Landscape & Strategic Context (3–5 items)
- NOT traditional policy/regulation news. Instead, surface items that matter for an early-stage AI founder:
  - **Market signals:** Shifts in where VCs are deploying capital, emerging consensus on AI product categories, pricing model experiments.
  - **Builder-relevant policy:** Only regulation that directly affects product design decisions (e.g. data privacy rules that constrain local-first vs. cloud architectures, API ToS changes from foundation model providers).
  - **Competitive dynamics:** New entrants, pivots, or shutdowns in adjacent spaces (personalization, cognitive AI, agent infrastructure).
- Skip macro governance news (UN summits, EU omnibus amendments, state-level regulation) unless it has direct product implications.

## Step 5 — Read active projects for relevance mapping

Read project files in `02_Projects/` with `status: active` to understand current vault priorities.
Use this context to annotate digest items with vault-project connections (e.g. "relevant to [[ProjectName]]", "relates to [[ResearchTopic]]").

## Step 6 — Write full digest

**Path:** `04_Knowledge/AI-Weekly/YYYY-WXX - AI Weekly Digest.md`

Create the `04_Knowledge/AI-Weekly/` directory if it doesn't exist.

**Frontmatter:**
```yaml
---
type: knowledge
kind: ai-weekly-digest
date: YYYY-MM-DD
week: YYYY-WXX
tags: [ai, weekly-digest, research, industry]
---
```

Use the actual date (today) and week identifier.

**Body structure:**

```markdown
# AI Weekly Digest · YYYY-WXX

## Academic Papers & Breakthroughs

### [Paper Title](link)
**Authors** — Author list
**TL;DR** — One-sentence summary of the paper.
**Why it matters** — Significance in the broader AI landscape.
**Relevance** — Connection to vault projects, if any.

(Repeat for 3–5 papers)

## Big Tech Updates

### Company Name
- **What happened** — Description of the announcement/release.
- **Why it matters** — Impact on the AI landscape.
- **Relevance** — Connection to vault projects, if any.

(Group by company, 5–8 items total)

## Early-Stage AI Startups

### Company Name — $Xm Pre-Seed/Seed/Series A
**What they're building** — One-liner on the product.
**Stage & raise** — Round details.
**Why it matters** — Competitive signal, market validation, or inspiration.
**Relevance** — Connection to vault projects, if any.

(5–8 items, pre-seed / seed / Series A only)

## Open-Source Model & Tool Releases

Top repos this week from [[GitHub Trending]] daily reports, ranked by cumulative stars:

### [owner/repo](link)
**What it is** — One-liner.
**Stars this week** — cumulative gained.
**Relevance** — Connection to vault projects.

(Top 5–8 repos)

### Weekly Open-Source Themes
- Bullet synthesis of patterns across the week's trending repos.

## Landscape & Strategic Context

### Item title
**What happened** — Description.
**Why it matters** — Implication for early-stage AI founders, product design, or competitive positioning.

(3–5 items: VC trends, builder-relevant policy, competitive dynamics — not macro governance)

## Synthesis & Outlook

- **Dominant theme this week** — One paragraph on the week's defining narrative.
- **Biggest signal for vault projects** — What matters most for active projects.
- **Watch next week** — 3–5 items to monitor in the coming week.
```

## Step 7 — Append summary to Weekly Review

Find `01_Execution/YYYY-WXX/Weekly Review.md` for the target week.

If it exists, insert a `### AI Weekly Digest` block **before** the `## Reflection` heading:

```markdown
### AI Weekly Digest
- **Papers:** [top paper title] — one-liner
- **Big Tech:** [top signal] — one-liner
- **Startups:** [top signal] — one-liner
- **Open-Source:** [top repo] — one-liner
- **Landscape:** [top signal] — one-liner
→ Full digest: [[YYYY-WXX - AI Weekly Digest]]
```

If Weekly Review doesn't exist yet, skip this step silently (the digest may be running before `/weekly-review`).

## Step 8 — Confirm

Print:
- The save path of the digest
- The target week (YYYY-WXX)
- One-sentence highlight per section (Academic, Big Tech, Startups, Open-Source, Policy)
- Whether the Weekly Review was updated