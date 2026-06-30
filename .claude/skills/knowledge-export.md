---
name: knowledge-export
description: Export structured knowledge to a target format. Supports RAG-ready JSONL chunks (for vector DBs), Markdown pages (for docs sites), and YAML frontmatter (for static site generators). Run after /knowledge-structure.
argument-hint: <session-id> [--format rag|markdown|yaml]
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /knowledge-export — Export to Target Format

Arguments: `$ARGUMENTS`

You are converting structured knowledge into the format the user actually needs. Parse `$ARGUMENTS` for:
- Session ID (required)
- `--format rag` | `--format markdown` | `--format yaml` (optional — ask if missing)

---

## Step 1: Load the session

1. Load `./knowledge-sessions/<session-id>.json`.
2. If status is not `structured`, tell the user to run `/knowledge-structure` first.
3. Read `./knowledge-sessions/<session-id>-structured.json` in full.

---

## Step 2: Determine export format

If `--format` was not passed in `$ARGUMENTS`, ask:

"Which export format do you need?
- **rag** — JSONL chunks for a vector database (OpenAI embeddings, Pinecone, Chroma, Weaviate, etc.)
- **markdown** — Separate `.md` files per knowledge object, ready for a docs site or wiki
- **yaml** — Markdown files with YAML frontmatter for static site generators (Hugo, Astro, Docusaurus)"

---

## Step 3: Export by format

### Format: `rag`

Output file: `./knowledge-sessions/<session-id>-export.jsonl`

Each line is a JSON object (one per knowledge item). Write in this schema — it is compatible with OpenAI, Pinecone, Chroma, and Weaviate metadata formats:

```json
{"id": "<object-id>", "text": "<chunk text>", "metadata": {"session_id": "<session-id>", "topic": "<topic>", "type": "<knowledge type>", "title": "<title>", "tags": ["<tag>"], "confidence": "<high|medium|low>", "priority": true|false, "expert": "<expert name/role>"}}
```

**Chunk text construction:**
```
[<TYPE>] <title>

<body>
```

For `procedure` type, render the steps numbered. For `rule` type, bold the condition. Keep each chunk under 500 tokens (roughly 375 words). If a body is longer, split it into multiple chunks with the same `id` suffixed `-part1`, `-part2`.

After writing, report: total chunks, estimated token count per chunk (rough), and which vector DB the format is compatible with.

---

### Format: `markdown`

Output directory: `./knowledge-sessions/<session-id>-export/`

Create one `.md` file per knowledge object, named `<object-id>.md`:

```markdown
# <title>

**Type:** <type> | **Confidence:** <confidence> | **Tags:** <tags as comma list>

<body>

---
*Source: <topic> — <expert> — Session <session-id>*
```

Also create an `index.md` in the export directory:

```markdown
# <topic> — Knowledge Base

**Expert:** <expert>
**Captured:** <date>
**Total items:** <count>

## Priority items
<bulleted list of priority object titles with links>

## All items by type

### Facts
- [<title>](<id>.md)

### Rules
...
```

---

### Format: `yaml`

Output directory: `./knowledge-sessions/<session-id>-export/`

Create one `.md` file per knowledge object with YAML frontmatter:

```markdown
---
title: "<title>"
type: <type>
tags: [<tags>]
confidence: <confidence>
priority: <true|false>
topic: "<topic>"
expert: "<expert>"
session: "<session-id>"
date: "<date>"
---

<body>
```

---

## Step 4: Update session and report

Update `./knowledge-sessions/<session-id>.json`: set `"status": "exported"`, add `"export_format": "<format>"`.

Tell the user:
- Where the export files are
- How many items were exported
- **For RAG format**: how to load the JSONL (brief note — e.g. "Each line is a dict; embed the `text` field, store the rest as metadata.")
- Next step: if they have more sessions, `/knowledge-merge` to combine them into a unified knowledge base. Otherwise they're done.
