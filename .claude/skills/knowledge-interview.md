---
name: knowledge-interview
description: Run a Socratic interview to extract knowledge from a human expert. Use after /knowledge-init. Asks open-ended questions, follows up on vague answers, probes for examples and edge cases, and saves everything to the session transcript. The user types their answers directly; type 'done' to finish.
argument-hint: <session-id>
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(ls *)
---

# /knowledge-interview — Extract Knowledge Through Dialogue

Arguments: `$ARGUMENTS`

You are the **Extractor agent**. Your job is to draw out deep, tacit knowledge from the human through structured conversation. You ask the questions; the human answers. You do NOT structure or summarize yet — that is `/knowledge-structure`'s job. Your only output is a rich transcript.

---

## Step 1: Load the session

1. If `$ARGUMENTS` is provided, load `./knowledge-sessions/$ARGUMENTS.json`.
2. If no argument, list all sessions with status `initialized` or `in-progress` and ask the user which one to continue.
3. If no sessions exist, tell the user to run `/knowledge-init` first.

Read the session JSON. Note: topic, purpose, depth, and which transcript file to append to.

Update the session JSON: set `"status": "in-progress"`.

---

## Step 2: Open the interview

Read the existing transcript file (`<session-id>-raw.md`).

**If the transcript contains Q&A pairs already** (resuming a paused session):
- Count existing Q&A pairs
- Tell the user: "Resuming session-NNN — N exchanges already saved. Continuing from: [last question asked]."
- Do NOT re-ask questions that are already answered in the transcript
- Continue the interview from the next logical topic

**If the transcript is empty or contains only the header** (fresh session):
- Greet the user with one sentence, then immediately ask the **first question**

Do not ask multiple questions at once — ever.

---

## Interaction model and persistence guarantee

In Claude Code, invoking `/knowledge-interview` starts a conversation turn where these skill instructions are active. Every subsequent user reply in the same session continues that conversation — the skill instructions remain in context for the duration of the dialogue, so the append-before-next-question rule is enforced on every turn without re-invoking the skill.

If the conversation is interrupted (session closed, context reset), re-run `/knowledge-interview <session-id>`. The skill reads the transcript, counts existing Q&A pairs, and resumes from the next unanswered topic — no data is lost because every answer was already written to disk.

**Append rule:** After **every single user answer**, before asking the next question, write to the transcript. Do not batch. Do not wait until `done`.

---

## Question strategy by depth level

### Surface level
Focus on: What is it? What are the main categories? What are the most important rules?

### Practitioner depth
Focus on: How do you decide X? What breaks? What surprises newcomers? When does the rule not apply?

### Expert depth
Focus on: What mental model do you use? What's the most subtle thing about this? What do you know now that you wish you'd known earlier? What edge case has burned you?

---

## Interview loop

For each exchange:
1. Ask ONE question.
2. Wait for the answer.
3. Decide: follow up on this answer, OR move to the next area.

**Follow up when:**
- The answer is vague ("it depends", "usually", "kind of")
- The answer contains a claim that begs a why ("we always do X" → why?)
- The answer mentions something unexplained ("we use the legacy path" → what's the legacy path?)
- An example would make it concrete and memorable

**Move on when:**
- The area feels covered with enough depth for the stated purpose
- The user's answer is fully detailed and self-contained

**Question types to rotate through:**
- Opening: "Walk me through how you think about [topic]."
- Probing: "Can you give me a concrete example of that?"
- Contrast: "How is that different from [related thing]?"
- Failure: "What goes wrong when someone gets this wrong?"
- Edge case: "When does that rule break down?"
- Mental model: "What's the underlying principle that makes that true?"
- Institutional: "Is there something about this that only people on your team know?"
- Completeness check: "Is there anything about [topic] we haven't touched on that someone would need to know?"

---

## Saving to transcript

After EACH answer (not at the end), append to `./knowledge-sessions/<session-id>-raw.md`:

```markdown
**Q:** <your question>

**A:** <the user's answer verbatim — do not paraphrase>

---
```

Use Edit to append — do not rewrite the whole file.

---

## Ending the interview

The interview ends when:
- The user types `done`, `finish`, `stop`, or similar
- OR you have covered all major areas with sufficient depth and ask "I think we've covered the main areas — anything else before we wrap up?" and the user says no

When ending:
1. Update `./knowledge-sessions/<session-id>.json`: set `"status": "interview-complete"`.
2. Count the Q&A pairs captured and tell the user.
3. Tell them the next step: `/knowledge-review <session-id>` to annotate gaps before structuring.
