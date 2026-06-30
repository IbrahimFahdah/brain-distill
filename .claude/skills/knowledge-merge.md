---
name: knowledge-merge
description: Merge multiple structured knowledge sessions into a single unified knowledge base. Deduplicates overlapping facts, resolves contradictions, builds a topic index, and outputs a merged export file. Run after /knowledge-export on two or more sessions.
argument-hint: [session-id1 session-id2 ...]
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(ls *)
  - Bash(mkdir *)
---

# /knowledge-merge — Merge Sessions into a Unified Knowledge Base

Arguments: `$ARGUMENTS`

You are combining multiple extraction sessions into one coherent knowledge base. This handles the case where knowledge was extracted across multiple sessions (different days, different experts, different subtopics) and needs to be unified before loading into a vector DB or docs site.

---

## Step 1: Determine which sessions to merge

If `$ARGUMENTS` contains session IDs (space-separated), use those.

Otherwise:
1. List all `./knowledge-sessions/session-*.json` files.
2. Show the user a table: session ID | topic | expert | status | item count.
3. Ask: "Which sessions should I merge? List the session IDs, or type 'all' to merge everything with status 'structured' or 'exported'."

Load the structured JSON for each selected session.

---

## Step 2: Deduplicate

Compare all knowledge objects across sessions using these rules in order:

1. **Exact title match** — treat as duplicate; compare bodies before merging.
2. **Same `type` + 2+ shared named entities** (function names, system names, metric names, product names) — likely duplicate; review both bodies.
3. **When in doubt, keep both** — a false negative (keeping a near-duplicate) is safer than a false positive (merging distinct knowledge). Mark uncertain cases with `"merge_confidence": "low"` in `merged_from`.

**Winner selection when merging a duplicate pair:**
- `high` confidence beats `medium` beats `low`
- Equal confidence: keep the object with the longer `body`
- Both bodies equal length: prefer the object that has a `source_quote`

Add a `merged_from` array to the kept object, listing all session IDs and original object IDs that were merged into it:
```json
"merged_from": [
  {"session": "session-001", "original_id": "session-001-007"},
  {"session": "session-002", "original_id": "session-002-003"}
]
```

If two versions **contradict** each other (one says X, one says not-X), flag it instead of silently picking one (see conflict handling below).

---

## Step 3: Resolve conflicts

Write every contradiction to `./knowledge-base/conflicts.md` using this format, then present them to the user one at a time:

```markdown
## Conflict: <title>
**Detected:** <ISO date>
**Type:** factual | procedural | policy

| | Session | Expert | Body excerpt (first 150 chars) |
|---|---|---|---|
| A | <session-id> | <expert> | <excerpt> |
| B | <session-id> | <expert> | <excerpt> |

**Resolution:** pending
**Notes:**
```

For each conflict, ask: "Which version is correct — A, B, or is there a nuance that reconciles them?"

- **A or B wins**: update `conflicts.md` resolution field; use the winning body in the merged object.
- **Both correct in different contexts**: split into two objects with distinct `tags` and a scoping phrase in the body (e.g. "When using X: …", "When using Y: …").
- **User unsure**: keep both objects, mark `"confidence": "low"`, leave resolution as `pending` in `conflicts.md`.

---

## Step 4: Build the merged knowledge base

Write `./knowledge-base/merged.json`:

```json
{
  "created_at": "<ISO date>",
  "sessions_merged": ["<session-id>", ...],
  "total_items": <count>,
  "topics": ["<topic>", ...],
  "knowledge": [
    {
      "id": "<new-global-id>",
      "source_sessions": ["<session-id>"],
      "type": "<type>",
      "title": "<title>",
      "body": "<body>",
      "tags": ["<tag>"],
      "related_ids": ["<global-id>"],
      "confidence": "<confidence>",
      "priority": true|false
    }
  ]
}
```

Global IDs format: `kb-<zero-padded-sequential-number>` (e.g. `kb-001`).

Before writing the merged file, build a mapping table of every original ID to its new global ID:
```
{ "session-001-007": "kb-001", "session-002-003": "kb-001", "session-001-012": "kb-002", ... }
```
Then substitute all `related_ids` values in every object using this table so cross-session links remain valid.

Also write `./knowledge-base/index.md` — a human-readable topic index:

```markdown
# Knowledge Base Index

**Sessions:** <list>
**Total items:** <count>
**Last merged:** <date>

## Topics
<for each topic>
### <topic>
- **Expert:** <expert>
- **Items:** <count>
- **Types:** <breakdown>

## Priority items (across all sessions)
<bulleted list of all priority items>

## All items by tag
<grouped list>
```

---

## Step 5: Export merged base

Ask: "Do you want to export the merged knowledge base now? (rag / markdown / yaml / skip)"

If not skip, apply the same export logic as `/knowledge-export` but reading from `./knowledge-base/merged.json` and writing to `./knowledge-base/export/`.

---

## Step 6: Report

Tell the user:
- Sessions merged
- Total items before and after deduplication
- Conflicts found and how resolved
- Where the merged files live
- How to add more sessions later: run `/knowledge-init` → interview pipeline → `/knowledge-merge` again (it will merge new sessions into the existing base)
