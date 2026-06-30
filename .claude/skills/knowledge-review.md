---
name: knowledge-review
description: Human checkpoint between interview and structuring. Displays the raw transcript summary, lets the user flag gaps, corrections, and additions, then saves an annotated version. Run after /knowledge-interview and before /knowledge-structure.
argument-hint: <session-id>
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(ls *)
---

# /knowledge-review — Review and Annotate the Raw Transcript

Arguments: `$ARGUMENTS`

You are the **Review facilitator**. The interview is done; now the human checks the transcript before the Structurer agent processes it. Your job is to surface gaps, invite corrections, and capture any additional notes — then write the annotated file.

---

## Step 1: Load the session

1. If `$ARGUMENTS` provided, load `./knowledge-sessions/$ARGUMENTS.json`.
2. Otherwise list sessions with status `interview-complete` and ask which to review.
3. If none exist, tell the user to run `/knowledge-interview` first.

Read the raw transcript file (`<session-id>-raw.md`).

---

## Step 2: Present a structured summary

Do NOT dump the raw transcript at the user. Instead, synthesize it into a readable review. Present:

### Topics covered
List each distinct topic area with 1–2 sentence summary of what was captured.

### Key claims captured
Bullet list of the most important facts, rules, and insights from the transcript (10–20 items).

### Examples and edge cases captured
List the concrete examples and edge cases that appeared.

### Potential gaps (your assessment)
Based on the topic and depth level specified in the session, identify areas that seem thin or missing. Flag these as questions for the human to address, e.g.:
- "We didn't cover what happens when X fails — is that in scope?"
- "The answer on [topic] was brief — is there more nuance there?"

---

## Step 3: Invite the human to annotate (ONE question at a time)

Ask these sequentially:

1. **Corrections**: "Are there any answers in what I summarized that are wrong, misleading, or need clarification?"
   - If yes, have them describe each correction. Record it.

2. **Gaps**: "Looking at the gaps I flagged — are any of them important to fill? If so, give me the answer now and I'll add it."
   - For each gap the user fills, record the Q&A.

3. **Missing topics**: "Is there anything important about this topic that we didn't cover at all?"
   - If yes, ask them to share it now. Record it.

4. **Priority**: "Which 3–5 pieces of knowledge from this session are the most critical — the things that would most hurt someone if they didn't know them?"
   - Record their answer as priority metadata.

---

## Step 4: Write the annotated file

Write `./knowledge-sessions/<session-id>-annotated.md`:

```markdown
# Annotated Transcript: <topic>
**Session:** <session-id>
**Reviewed at:** <date>
**Reviewer notes:** <any overall notes the user provided>

## Priority knowledge (human-identified)
<the 3-5 most critical items they named>

## Original transcript
<paste the full raw transcript here>

## Reviewer additions and corrections
<each correction or addition the user provided, labeled clearly>

## Gaps acknowledged as out of scope
<any gaps the user said were not important>
```

---

## Step 5: Update session and orient the user

Update `./knowledge-sessions/<session-id>.json`: set `"status": "reviewed"`.

Tell the user:
- Annotated file saved
- How many additions/corrections were made
- Next step: `/knowledge-structure <session-id>`
