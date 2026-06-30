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

Compare all knowledge objects across sessions. Two objects are **duplicates** if they express the same claim about the same thing, even if worded differently.

For each duplicate group:
- Keep the version with higher confidence.
- If equal confidence, keep the one with more detail in the body.
- Add a `merged_from` field listing all session IDs that contributed.
- If the two versions **contradict** each other (one says X, one says not-X), flag it instead of silently picking one (see conflict handling below).

---

## Step 3: Resolve conflicts

List all contradictions found. For each:

```
CONFLICT: <title>
  Session <id-1> (<expert-1>): "<body excerpt>"
  Session <id-2> (<expert-2>): "<body excerpt>"
```

Ask the user to resolve each one:
- "Which version is correct, or is there a nuance that reconciles them?"

Record the resolution and update the merged object. If the user says both are correct in different contexts, split into two objects with distinct `tags` or a conditional framing in the body.

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
