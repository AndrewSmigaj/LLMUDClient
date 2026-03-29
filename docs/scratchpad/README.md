# docs/scratchpad/ — Working Architecture Documents

This folder holds design documents that are being developed but have NOT been approved for inclusion in the main docs.

## Rules

1. All new architectural proposals start here, not in main docs.
2. Use the template below for every new scratchpad doc.
3. Only Emily can approve promotion to main docs.
4. When promoted: merge content into the target doc, delete this file, update INDEX.md.
5. When rejected: delete with a descriptive commit message. Git history preserves it.
6. Scratchpad docs older than 30 days without activity should be reviewed for relevance — ask Emily whether to continue, promote, or delete.

## Naming Convention

`YYYY-MM-DD-short-name.md` — date is creation date, lowercase-hyphenated name.

Examples:
- `2026-03-28-memory-consolidation-redesign.md`
- `2026-04-01-tick-speed-research.md`

## Template

```markdown
---
title: [Descriptive Title]
status: draft | review
created: YYYY-MM-DD
target_doc: [Which main doc this will merge into, e.g., AI_SYSTEM_DESIGN.md]
affects: [List of other docs that would need updates if this is approved]
summary: [1-2 sentence summary of what this proposes]
---

## Problem

What problem does this solve? Why is the current design insufficient?

## Proposal

The actual design content. Write this at the quality level of the target doc —
when promoted, this section gets merged directly.

## Alternatives Considered

What else was considered and why it was rejected.

## Impact

What existing docs, code, or decisions change if this is adopted?
List specific sections that would need updating.

## Open Questions

Unresolved issues that need discussion before promotion.
```

## Lifecycle

```
Create (draft) → Iterate (draft) → Set status to "review" → Emily approves
                                                                │
                                              ┌─────────────────┴──────────────────┐
                                              │                                    │
                                        Promote:                             Reject:
                                        merge into target doc,               delete with descriptive
                                        delete scratchpad file,              commit message
                                        update INDEX.md,                     (git history preserves)
                                        follow Change Propagation
                                        for `affects` docs
```

## Why This Exists

Claude plans have meaningless auto-generated names and are hard to find or read. Scratchpad docs are:
- **Named** — descriptive filenames you can find later
- **In the repo** — versioned, diffable, reviewable
- **Structured** — frontmatter tells any AI session what this is and where it goes
- **Gated** — nothing reaches the main docs without Emily's explicit approval
