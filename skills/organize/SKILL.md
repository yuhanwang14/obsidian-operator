---
name: organize
description: "Scan the Obsidian vault for misplaced files, missing frontmatter, and notes that should be split or merged. TRIGGER when the user wants to tidy, organize, audit, or clean up the vault structure. Signal phrases: 'organize my vault', 'check for misplaced files', 'audit vault structure', 'tidy up notes', 'what's out of place'. Also triggers for /organize. Produces recommendations only — does not move or edit files. NOT for file management outside the vault, desktop cleanup, or code organization."
version: 1.0.0
author: Yuhan Wang
license: MIT
tags: [obsidian, vault, maintenance, organization]
---

Scan the vault for recently modified or newly created files.

For each file that is misplaced, unstructured, or missing standard formatting:
1. **Suggest the correct folder** based on the vault structure in CLAUDE.md
2. **Flag missing frontmatter** on project or execution notes that need it
3. **Identify notes that should be split or merged**

Do not move or edit files — only surface recommendations for me to act on.
