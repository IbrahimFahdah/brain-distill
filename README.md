# Knowledge Extraction Skills for Claude Code

A chain of six Claude Code skills that interview a human expert and convert their tacit knowledge into structured, RAG-ready output. Inspired by [Matt Pocock's skills concept](https://github.com/mattpocock/skills).

```
/knowledge-init
       ↓
/knowledge-interview   ← repeatable
       ↓
/knowledge-review
       ↓
/knowledge-structure
       ↓
/knowledge-export
       ↓
/knowledge-merge       ← optional, combines multiple sessions
```

---

## What it does

Most expert knowledge lives in people's heads — the reasoning behind decisions, the edge cases that burned them, the mental models they use. This pipeline extracts it through dialogue and converts it into typed, self-contained knowledge objects you can load into a vector database, publish as documentation, or use for onboarding.

The key design: the **Extractor** (interview) and **Structurer** (formatting) are separate skills. Trying to structure while interviewing kills nuance.

---

## Installation

Clone this repo into your project and Claude Code will discover the skills automatically:

```bash
git clone https://github.com/IbrahimFahdah/brain-distill .claude/skills/knowledge-extraction
```

Or copy the `.claude/skills/` folder into any existing project:

```bash
cp -r .claude/skills/* your-project/.claude/skills/
```

No dependencies. No API keys beyond your Claude Code session.

---

## Usage

Run these from your project directory in sequence. Each skill tells you the next step when it finishes.

### 1. `/knowledge-init`

Creates the session scaffold. Asks:
- What topic/domain are you extracting?
- Who is the expert?
- What is the output for? (RAG, docs, onboarding)
- How deep should we go? (surface / practitioner / expert)

```
/knowledge-init kubernetes networking
```

Creates `./knowledge-sessions/session-001.json` and the blank transcript file.

---

### 2. `/knowledge-interview <session-id>`

Runs a Socratic dialogue. The skill asks questions one at a time; you answer as yourself or relay the expert's answers. It follows up on vague answers, asks for examples, probes edge cases. Type `done` when finished.

```
/knowledge-interview session-001
```

Saves every Q&A exchange to `./knowledge-sessions/session-001-raw.md` after each answer. You can pause and resume — it picks up where it left off.

---

### 3. `/knowledge-review <session-id>`

Human checkpoint before structuring. The skill:
- Summarises what was captured (not a raw dump)
- Flags potential gaps
- Asks you to add corrections, missing topics, and priority items

```
/knowledge-review session-001
```

Saves `./knowledge-sessions/session-001-annotated.md`.

---

### 4. `/knowledge-structure <session-id>`

The Structurer agent reads the annotated transcript and converts it into typed knowledge objects:

| Type | What it captures |
|---|---|
| `fact` | Declarative claims about the domain |
| `concept` | Terms with domain-specific meanings |
| `rule` | Decision rules: when X, do Y because Z |
| `procedure` | Ordered steps to accomplish something |
| `example` | Concrete illustrations linked to a parent object |
| `edge_case` | Situations where the normal rule breaks down |
| `mental_model` | How the expert reasons through new situations |
| `warning` | Common failure modes with consequences |

```
/knowledge-structure session-001
```

Each object is self-contained — no pronouns that require prior context, no "the user said" framing. Saves `./knowledge-sessions/session-001-structured.json`.

---

### 5. `/knowledge-export <session-id> --format <rag|markdown|yaml>`

Converts structured knowledge to your target format.

```
/knowledge-export session-001 --format rag
```

**`rag`** — JSONL, one chunk per line. Compatible with OpenAI embeddings, Pinecone, Chroma, Weaviate. Each chunk has `text` (embed this) and `metadata` (store this).

**`markdown`** — One `.md` file per knowledge object plus an `index.md`. Ready for a docs site or wiki.

**`yaml`** — Markdown with YAML frontmatter. Works with Hugo, Astro, Docusaurus, and similar static site generators.

---

### 6. `/knowledge-merge` (optional)

Combines multiple sessions into a single unified knowledge base. Deduplicates overlapping facts, flags contradictions for human resolution, and builds a topic index.

```
/knowledge-merge session-001 session-002 session-003
```

Output: `./knowledge-base/merged.json` and `./knowledge-base/index.md`.

---

## Output structure

```
your-project/
└── knowledge-sessions/
    ├── session-001.json              ← session metadata and status
    ├── session-001-raw.md            ← verbatim interview transcript
    ├── session-001-annotated.md      ← transcript + reviewer additions
    ├── session-001-structured.json   ← typed knowledge objects
    └── session-001-export.jsonl      ← RAG-ready chunks (or .md files)
knowledge-base/
    ├── merged.json                   ← unified knowledge base
    └── index.md                      ← human-readable topic index
```

---

## Example structured object

```json
{
  "id": "session-001-007",
  "type": "edge_case",
  "title": "Retry rule does not apply to payment endpoints",
  "body": "Standard retry-with-backoff must NOT be applied to payment processing endpoints. Retrying a charge after a network timeout can result in a double-billing if the first request succeeded but the response was lost. Use idempotency keys instead.",
  "tags": ["payments", "reliability", "retries"],
  "confidence": "high",
  "priority": true,
  "source_quote": "we got burned by this in 2022 — customer got charged twice because the retry hit after a timeout..."
}
```

---

## Design notes

**Why two separate agents?** Interviewing and structuring are fundamentally different cognitive tasks. An agent that tries to structure while it interviews will prematurely close on answers, stop following up, and miss the institutional knowledge that only surfaces with persistent probing.

**Why verbatim answers in the transcript?** The raw transcript preserves things the structuring step might miss. If the structured output ever looks wrong, you can trace it back to what the expert actually said.

**Why self-contained object bodies?** RAG chunks get retrieved individually, without surrounding context. An object body that says "as mentioned above" or "unlike the previous approach" is useless when retrieved in isolation.

---

## Skills

```
.claude/skills/
├── knowledge-init.md
├── knowledge-interview.md
├── knowledge-review.md
├── knowledge-structure.md
├── knowledge-export.md
└── knowledge-merge.md
```

---

## License

MIT
