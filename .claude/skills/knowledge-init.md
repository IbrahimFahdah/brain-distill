---
name: knowledge-init
description: Start a new knowledge extraction session. Use when the user wants to capture expertise, document know-how, build a knowledge base, or begin an interview with a subject matter expert. Creates the session scaffold for the /knowledge-interview → /knowledge-structure → /knowledge-export pipeline.
argument-hint: [topic]
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(echo *)
---

# /knowledge-init — Start a Knowledge Extraction Session

Arguments: `$ARGUMENTS`

You are setting up a new knowledge extraction session. This is the first step in the pipeline:

```
/knowledge-init → /knowledge-interview → /knowledge-review → /knowledge-structure → /knowledge-export
```

---

## Step 1: Gather session details (ask ONE question at a time, wait for each answer)

If `$ARGUMENTS` is provided, use it as the topic and skip question 1.

1. **Topic/domain**: "What topic or domain are we extracting knowledge from? Be as specific as you can — e.g. 'Kubernetes networking troubleshooting' not just 'DevOps'."

2. **Expert**: "Who is the knowledge holder? (Name/role — this is for session metadata, not shared externally.)"

3. **Purpose**: "What will this knowledge be used for?"
   - A) RAG / semantic search (chunks go into a vector DB)
   - B) Internal documentation (readable Markdown pages)
   - C) Onboarding material (structured guides for new team members)
   - D) Other — describe it

4. **Depth**: "How deep should we go?"
   - A) Surface level — key concepts and rules of thumb
   - B) Practitioner depth — the reasoning behind decisions, edge cases, gotchas
   - C) Expert depth — mental models, failure modes, subtle invariants, institutional knowledge

---

## Step 2: Create session files

1. Check if `./knowledge-sessions/` exists. If not, create it.

2. Determine the next session ID:
   - List `./knowledge-sessions/session-*.json` files
   - Extract the numeric suffix from each filename (e.g. `session-003.json` → `3`)
   - Next ID = **highest suffix found + 1**, zero-padded to 3 digits (e.g. `session-004`)
   - If no sessions exist, start at `001`
   - Do NOT use file count — deleted sessions would cause ID collisions

3. Normalize the user's answers to schema enum values before writing:

   | User answer | `purpose` value to write |
   |---|---|
   | A / RAG / semantic search / vector DB | `rag` |
   | B / documentation / docs / internal docs | `docs` |
   | C / onboarding / guides | `onboarding` |
   | D / anything else | `other` |

   | User answer | `depth` value to write |
   |---|---|
   | A / surface / key concepts / rules of thumb | `surface` |
   | B / practitioner / reasoning / edge cases | `practitioner` |
   | C / expert / mental models / failure modes | `expert` |

4. Write `./knowledge-sessions/<session-id>.json`:

```json
{
  "session_id": "<session-id>",
  "created_at": "<ISO date>",
  "topic": "<topic from user>",
  "expert": "<expert name/role>",
  "purpose": "rag | docs | onboarding | other",
  "depth": "surface | practitioner | expert",
  "status": "initialized",
  "files": {
    "raw_transcript": "<session-id>-raw.md",
    "annotated": "<session-id>-annotated.md",
    "structured": "<session-id>-structured.json",
    "export": "<session-id>-export.jsonl"
  }
}
```

5. Write `./knowledge-sessions/<session-id>-raw.md` with just a header:

```markdown
# Knowledge Extraction: <topic>
**Session:** <session-id>
**Expert:** <expert>
**Date:** <date>
**Purpose:** <purpose>
**Depth:** <depth>

---

<!-- Interview transcript below — appended during /knowledge-interview -->
```

---

## Step 3: Confirm and orient the user

Tell the user:
- Session ID created (e.g. `session-001`)
- Where files live (`./knowledge-sessions/`)
- The exact next command to run: `/knowledge-interview <session-id>`
- Brief note: the interview will ask questions — the user answers as themselves or relays the expert's answers. They type `done` when finished.
