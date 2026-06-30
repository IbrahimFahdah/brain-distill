---
name: knowledge-structure
description: Transform the reviewed interview transcript into structured, typed knowledge objects. Run after /knowledge-review. Produces a JSON file with atomic facts, concept definitions, decision rules, procedures, examples, and edge cases — ready for /knowledge-export.
argument-hint: <session-id>
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
---

# /knowledge-structure — Structure the Knowledge

Arguments: `$ARGUMENTS`

You are the **Structurer agent**. The human interview is done and reviewed. Now you read the annotated transcript and convert it into typed, structured knowledge objects. You do NOT interview or ask clarifying questions — you work from what is in the transcript. If something is ambiguous, note it in the object rather than guessing.

---

## Step 1: Load the session

1. If `$ARGUMENTS` provided, load `./knowledge-sessions/$ARGUMENTS.json`.
2. Otherwise list sessions with status `reviewed` and ask which to structure.
3. If none exist, tell the user to run `/knowledge-review` first.

Read the annotated transcript file (`<session-id>-annotated.md`) in full.

---

## Step 2: Extract knowledge objects

Read the entire annotated transcript. For each distinct piece of knowledge, classify it into one of these types and extract it as a structured object:

### Knowledge types

**`fact`** — A factual claim about the world, system, or domain. Stable, declarative.
> Example: "Kubernetes readiness probes block traffic to a pod until it passes; liveness probes restart the pod if it fails."

**`concept`** — A term or abstraction with a specific meaning in this domain. Often misunderstood or domain-specific.
> Example: "In this codebase, 'settlement' means the point at which a transaction is irrevocably committed to the ledger, not just confirmed."

**`rule`** — A decision rule, heuristic, or policy. Has a condition and a recommended action.
> Example: "If a service has > 3 dependencies, always add a circuit breaker. If it has only one, a retry with backoff is sufficient."

**`procedure`** — A sequence of steps to accomplish something. Ordered.
> Example: "To roll back a migration: 1) put the service in maintenance mode, 2) run the rollback script, 3) verify row counts, 4) re-enable traffic."

**`example`** — A concrete illustration of a fact, concept, or rule. Linked to its parent.

**`edge_case`** — A situation where the normal rule breaks down, or a gotcha the expert has encountered.
> Example: "The retry rule does NOT apply to payment processing endpoints — retrying a charge can double-bill."

**`mental_model`** — A way of thinking about the domain that the expert uses to reason through new situations.
> Example: "Think of Kafka topics as an infinite tape. Consumers don't consume from the topic — they maintain their own read position."

**`warning`** — Something that commonly goes wrong, with consequences. Priority metadata.

---

## Step 3: Write the structured JSON

Write `./knowledge-sessions/<session-id>-structured.json`:

```json
{
  "session_id": "<session-id>",
  "topic": "<topic>",
  "expert": "<expert>",
  "structured_at": "<ISO date>",
  "purpose": "<purpose>",
  "priority_items": ["<id>", "<id>"],
  "knowledge": [
    {
      "id": "<session-id>-001",
      "type": "fact | concept | rule | procedure | example | edge_case | mental_model | warning",
      "title": "<short title — becomes a chunk heading in RAG>",
      "body": "<the knowledge itself, written as a standalone paragraph — no pronouns that require prior context>",
      "tags": ["<tag>", "<tag>"],
      "related_ids": ["<id of related object in this session>"],
      "confidence": "high | medium | low",
      "ambiguity_note": "<optional — note if the source was unclear or if an assumption was made>",
      "source_quote": "<the verbatim answer this was extracted from, truncated to ~200 chars>"
    }
  ]
}
```

Rules for writing `body`:
- Must be **self-contained**: a reader with no prior context can understand it.
- No "the user said" or "according to the expert" framing — just the knowledge.
- For rules: include condition AND action ("When X, do Y because Z").
- For procedures: include numbered steps.
- For edge cases: include the trigger condition and what breaks.
- Write at the depth level specified in the session metadata.

---

## Step 4: Quality check

After writing, re-read the structured JSON and verify:
- Every significant claim from the annotated transcript appears in at least one object
- Priority items (from the review step) are included and marked in `priority_items`
- No object body requires reading another object to understand it
- Tags are consistent (e.g. don't use both "k8s" and "kubernetes")

If you find gaps, add the missing objects.

---

## Step 5: Update session and orient the user

Update `./knowledge-sessions/<session-id>.json`: set `"status": "structured"`.

Report to the user:
- Total knowledge objects created, broken down by type (e.g. "8 facts, 3 rules, 4 edge_cases...")
- Any ambiguity notes that the user should clarify
- Next step: `/knowledge-export <session-id>`
