---
name: plea
description: Use when starting a new feature, project, or significant code change and requirements are unclear. Runs an adaptive binary-question interview before any code is written, then outputs a concrete PLAN.md. Triggers on /plea, "clarify requirements", "help me plan", "what should I build", "before we start coding", or when the user's request is ambiguous and would benefit from structured requirements gathering.
---

# Method — Adaptive Interview for Code Planning

EXECUTE THIS PROTOCOL EXACTLY. Follow each phase in order. Do not skip ahead.

## MANDATORY RULES

1. **ONE QUESTION AT A TIME.** Ask exactly ONE binary question per AskUserQuestion call. Never batch multiple questions.
2. **YES/NO OPTIONS ONLY.** Every AskUserQuestion call MUST use `options: ["Yes", "No"]`. No other options. No freeform text prompts.
3. **USE AskUserQuestion TOOL.** Every question MUST use the tool. Never print questions as text.
4. **NO ASSUMPTIONS.** Never assume answers. Never offer defaults. Never skip questions.
5. **NO TABLES.** No comparison tables, no pros/cons lists, no multi-choice.
6. **WAIT AFTER EACH QUESTION.** Call AskUserQuestion, then STOP. Do not continue until the user responds.
7. **SHOW PROGRESS.** Every question starts with `[{N}/{~total}]` counter.

### Question Format

Every question follows this exact pattern:

```
question: "[{N}/{~total}] {Single binary question}?"
options: ["Yes", "No"]
```

Example:
```
question: "[3/~15] Does this need persistent storage?"
options: ["Yes", "No"]
```

If the user responds with text instead of clicking Yes/No, incorporate the information and adjust. This is valuable signal — don't reject it.

---

## Phase 1: SCAN

Gather context SILENTLY. Do not narrate what you're reading. Read these files (skip missing):

1. `PLAN.md` in project root
2. `.plea/` directory for prior sessions
3. `CLAUDE.local.md` for stored decisions
4. `CLAUDE.md` / `AGENTS.md` for project conventions

Scan the codebase:
- Small project (<50 files): read directory listing
- Large project: use `Explore` subagent for tech stack summary

If prior session found:
```
question: "Found session from {date}: '{request}'. Resume?"
options: ["Yes", "No"]
```

Store facts internally. Move to Phase 2.

---

## Phase 2: PREFILL

If project is empty (no source files), SKIP to Phase 3.

If codebase exists, present facts and confirm:
```
question: "I see: {stack}, {db}, {auth}, {tests}, {N} files. All correct?"
options: ["Yes", "No"]
```

If No: ask what's wrong via AskUserQuestion (one correction at a time).

Move to Phase 3.

---

## Phase 3: INTERVIEW

One question at a time. **After every answer, re-assess the full picture before asking the next question.** This is the core of the clinical method — not a script, a diagnosis.

### Step 3.1: Depth

```
question: "Interview depth for '{request}'?"
options: ["Quick (~5 questions, ~1 min)", "Standard (~15 questions, ~3 min)", "Thorough (30+ questions, ~8 min)"]
```

This is the ONE exception where more than two options are allowed — it's a depth setting, not a design question.

### Step 3.2: Load Taxonomy

Read `references/question-axes.md`. Use it as a reference, not a script.

### Step 3.3: Interview Loop — The Clinical Method

The loop has TWO mandatory steps per cycle: ASK, then RE-ASSESS. Never skip re-assessment.

Start a counter at 1. REPEAT until depth target reached or user says "enough":

#### A. ASK — one question

Pick the single most diagnostic question right now. "Most diagnostic" = the one whose answer would change the plan the most. Use the taxonomy axes as inspiration, but do NOT follow them linearly. After re-assessment, the most important question might jump to a completely different axis.

```
question: "[{N}/{~total}] {question}?"
options: ["Yes", "No"]
```

**STOP. Wait for response.**

#### B. RE-ASSESS — update the full picture and show the delta

This is the critical step. After EVERY answer, before picking the next question, you MUST:

1. **Record** the answer.
2. **Update your mental model** of what's being built. What do you now know? What hypotheses are eliminated? What's the current shape of the solution?
3. **Check for contradictions** against ALL previous answers. If found:
   ```
   question: "Conflict: you said '{A}' but also '{B}'. Keep which?"
   options: ["Keep first", "Keep second"]
   ```
4. **Eliminate branches.** Which axes or questions are now irrelevant? Remove them.
5. **Unlock branches.** Did the answer reveal new complexity? Add deeper questions.
6. **Show the delta.** This is what makes the adaptive tree visible. Print a brief status line BEFORE asking the next question:
   - If questions were eliminated: `{N} skipped ({reason}) · ~{remaining} remaining`
   - If questions were added: `+{N} unlocked ({reason}) · ~{remaining} remaining`
   - If no change: `~{remaining} remaining`
   - Examples:
     - `3 skipped (no database = skip data axis) · ~9 remaining`
     - `+2 unlocked (auth details needed) · ~14 remaining`
     - `4 skipped (existing framework covers this) · ~6 remaining`
   The delta shows the developer their answer CHANGED the tree. This builds trust and makes the adaptive nature the star of the show.
7. **Recalculate priority.** What is now the MOST informative question to ask? It may be on a completely different axis than the last one.
8. **If free text was given** instead of Yes/No: extract all facts from it. Mark any questions it implicitly answered. Show: `Got it — that also answers {N} questions. ~{remaining} remaining.`

Only AFTER completing re-assessment → go back to step A.

### Step 3.4: Safety Valves

**All-yes (>80% after 8+ questions):**
```
question: "Most answers are yes — large scope. Re-examine the biggest decision: {highest_impact_question}. Still yes?"
options: ["Yes", "No"]
```

**All-no (>80% after 8+ questions):**
```
question: "Most answers are no — am I asking wrong questions? Can you describe what this needs in one sentence?"
```
(This is the ONE exception where freeform text is requested.)

**"Enough" / "stop":** Move to Phase 4 immediately with answers collected so far.

---

## Phase 4: SYNTHESIZE

Generate PLAN.md SILENTLY. No commentary.

1. Read `references/plan-template.md`.
2. Final contradiction check.
3. Fill every section from collected answers + SCAN context.
4. If PLAN.md exists:
   ```
   question: "PLAN.md exists. Overwrite?"
   options: ["Yes", "No"]
   ```
   If No → write PLAN-2.md.
5. Write file.

---

## Phase 5: PERSIST

### 5.1: Session File

Create `.plea/sessions/{YYYY-MM-DD-HHmmss}.json`:

```json
{
  "request": "",
  "timestamp": "",
  "depth": "",
  "total_questions": 0,
  "questions": [
    { "id": 1, "axis": "", "text": "", "answer": "", "source": "user" }
  ],
  "contradictions": [],
  "plan_file": "PLAN.md"
}
```

### 5.2: CLAUDE.local.md

Append:
```markdown
## plea decisions ({YYYY-MM-DD})
- Task: {title}
- Plan: see PLAN.md
- Key decisions: {top 5, one line each}
```

### 5.3: Gitignore

```
question: "Add .plea/ to .gitignore?"
options: ["Yes", "No"]
```

---

## Phase 6: OFFER

```
question: "PLAN.md written ({N} decisions, {M} files). Start building?"
options: ["Yes", "No"]
```

If Yes: suggest execution skills if available.
If No: end session.

---

## Language

- Detect from user input. Interview in user's language.
- All files stay in English.

## Error Recovery

| Situation | Action |
|-----------|--------|
| No request | Ask: "What do you want to build?" |
| Codebase too large | Top-level only |
| Write fails | Output as text |
| User abandons | Save partial with `"complete": false` |
| Prior session, different request | Ask: "Start fresh?" |
