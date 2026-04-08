---
name: clarify
description: Use when starting a new feature, project, or significant code change and requirements are unclear. Runs an adaptive binary-question interview before any code is written, then outputs a concrete PLAN.md. Triggers on /clarify, "clarify requirements", "help me plan", "what should I build", "before we start coding", or when the user's request is ambiguous and would benefit from structured requirements gathering.
---

# Clarify — Adaptive Interview for Code Planning

Interview the developer with binary questions to extract architectural decisions. Each question eliminates a branch of the decision tree. Output: a concrete PLAN.md with files, changes, dependencies, and implementation order.

**Method:** Adaptive clinical interview. Not Socratic, not brainstorming. Modeled on differential diagnosis — each question must narrow the solution space.

## IMPORTANT: Execution Rules

- Ask questions via `AskUserQuestion` — never assume answers
- Binary questions only. Decompose "A or B?" into two binary questions
- If a fact is derivable from the codebase or memory, confirm it — don't ask
- Every question must eliminate at least one decision branch. If it wouldn't change the plan, don't ask it
- Accept free-text answers — if the user writes more than y/n, incorporate the information
- The user can say "enough" at any batch boundary to skip to synthesis

---

## Phase 1: SCAN

Gather context before asking anything. Read these in order (skip missing files silently):

1. `PLAN.md` in project root — if exists, this is a re-run. Load previous decisions as defaults.
2. `.clarify/` directory — check for prior sessions. If found with same project, offer to resume.
3. `CLAUDE.local.md` — check for stored decisions from prior clarify sessions.
4. `CLAUDE.md` / `AGENTS.md` — project conventions and constraints.

Then scan the codebase:

- Use the `Explore` subagent type (or read files directly if project is small):
  ```
  Summarize this project: tech stack, frameworks, languages, file count, directory structure,
  test framework, auth mechanism, database, deployment target. Be concise — bullet points only.
  ```
- If project is empty (no source files), note this — questions will be broader.

**Session Resumption:** If `.clarify/sessions/` contains a session file for this project:
```
Found interrupted session from {date}: "{original_request}"
Resume where you left off? [Y/n]
```
If yes, load all prior answers and continue from the last unanswered question. Skip to Phase 3.

Store all discovered facts as a structured list for Phase 2.

---

## Phase 2: PREFILL

Present derived facts for confirmation. Do NOT ask questions the codebase already answers.

Format:
```
Based on your project:
  1. Stack: {languages, frameworks}
  2. Database: {type or "none detected"}
  3. Auth: {mechanism or "none detected"}
  4. Testing: {framework or "none detected"}
  5. Deployment: {target or "unknown"}
  6. Size: {file count} files, ~{LOC} lines of code

All correct? [Y/n]
```

Use `AskUserQuestion` with options: "All correct", "Some corrections needed".

If corrections needed, ask: "Which items need correction? (enter numbers, e.g., 2,4)"
Then ask for the correct value for each flagged item.

If project is empty, skip PREFILL entirely — there's nothing to confirm.

---

## Phase 3: INTERVIEW

This is the core phase. Generate and ask adaptive binary questions in batches.

### Step 3.1: Determine Depth

Auto-estimate from request complexity:

| Signal | Suggested Depth |
|--------|----------------|
| Request names specific files or functions | compact (~5 questions) |
| Request describes a single feature | standard (~15 questions) |
| Request describes a new project or major architecture change | thorough (30+ questions) |
| Request is vague or open-ended | standard, then adjust |

Present the suggestion:
```
Estimated complexity: {standard}. Interview depth: ~{15} questions.
Override? [compact / standard / thorough / accept]
```

### Step 3.2: Load Question Taxonomy

Read `references/question-axes.md` from this skill's directory. This provides 10+ question axes with seed questions at each depth level.

### Step 3.3: Run Interview Loop

**For each batch (5 questions per batch):**

1. **Select questions:** Pick the 5 most relevant unasked questions across axes, prioritizing:
   - Higher-priority axes first (Scope > Data > Auth > ...)
   - Questions whose answers would eliminate the most branches
   - Axes flagged for deeper exploration by previous answers
   - Skip questions whose answers are already known from PREFILL

2. **Present batch:** Use `AskUserQuestion` with this format:
   ```
   [{current}/{~total}] Questions:

   1. [scope] Does this change cross multiple modules? [y/n]
   2. [data] Does this need persistent storage? [y/n]
   3. [auth] Does this require authentication? [y/n]
   4. [api] Does this expose an API? [y/n]
   5. [testing] Are tests required? [y/n]

   Answer: (e.g., "y n y y n" or "1:y 2:n 3:y 4:y 5:n")
   ```

3. **Parse responses:** Accept these formats:
   - Shorthand: `y n y y n` (space-separated, matches question order)
   - Numbered: `1:y 2:n 3:yes 4:no 5:y`
   - Natural language: parse intent from free text
   - Partial: if only some answered, ask about the rest

4. **After each batch:**
   - Check for contradictions: load `references/contradiction-rules.md` and cross-reference all answers so far
   - If contradiction found: present it immediately, ask user to resolve before continuing
   - Evaluate which axes need deeper exploration based on answers
   - Update progress estimate (total may change as axes open/close)
   - If depth target reached, move to Phase 4

5. **Adaptive adjustments:**
   - If an answer reveals unexpected complexity → add deeper questions for that axis
   - If an answer eliminates an entire axis → remove its remaining questions from the queue
   - If user provides free-text instead of y/n → extract all facts from the text, skip questions already answered by it

### Step 3.4: Safety Checks

**All-yes pattern (>80% yes):**
```
You've said yes to most questions — this is shaping up to be a large scope.
Let me re-confirm the 2 highest-impact decisions:
```
Then re-ask the 2 questions with the largest plan impact.

**All-no pattern (>80% no):**
```
Most answers are "no" — I may be asking the wrong questions.
In a sentence or two, what does this feature actually need?
```
Incorporate the free-text answer and restart question selection.

**"Enough" command:** If user says "enough", "stop", "that's enough", or similar at any point → immediately move to Phase 4 with answers collected so far.

---

## Phase 4: SYNTHESIZE

Generate the PLAN.md.

1. Read `references/plan-template.md` from this skill's directory for the output format.

2. Run a **final contradiction check** across all answers before writing.

3. Fill the template:
   - **Summary:** Synthesize from all answers — what's being built, key approach, why
   - **Decisions table:** All questions and answers, in order asked
   - **Files:** Predict specific file paths using the project's actual structure from SCAN
   - **Implementation order:** Logical phases based on dependency order
   - **Non-functional requirements:** Extracted from relevant answers (rate limiting, accessibility, performance budgets)
   - **Open questions:** Anything unresolved or deferred

4. **Handle existing PLAN.md:**
   - If PLAN.md exists in project root, ask: "Overwrite existing PLAN.md or create PLAN-2.md?"
   - Use `AskUserQuestion` with those two options

5. Write PLAN.md to project root using the Write tool.

---

## Phase 5: PERSIST

Save session data for future reference and resumption.

### 5.1: Session File

Create `.clarify/sessions/` directory if needed. Write session JSON:

```json
{
  "request": "{original user request}",
  "timestamp": "{ISO 8601}",
  "depth": "{compact|standard|thorough}",
  "total_questions": {N},
  "total_batches": {N},
  "codebase_summary": "{from SCAN phase}",
  "prefill": {
    "facts": ["{list of confirmed facts}"],
    "corrections": ["{any corrections made}"]
  },
  "questions": [
    {
      "id": 1,
      "batch": 1,
      "axis": "{axis name}",
      "text": "{question text}",
      "answer": "{yes|no|free-text value}",
      "source": "{user|prefill_confirmed|derived}"
    }
  ],
  "contradictions": [
    {
      "question_a": {id},
      "question_b": {id},
      "severity": "{hard|tension}",
      "resolution": "{which answer was kept or explanation}"
    }
  ],
  "plan_file": "PLAN.md"
}
```

Filename: `{YYYY-MM-DD-HHmmss}.json`

### 5.2: CLAUDE.local.md Update

Append to `CLAUDE.local.md` (create if not exists):

```markdown
## clarify decisions ({YYYY-MM-DD})
- Task: {short title from PLAN.md}
- Plan: see PLAN.md
- Key decisions:
  - {decision 1}: {value}
  - {decision 2}: {value}
  - {decision 3}: {value}
  - {decision 4}: {value}
  - {decision 5}: {value}
```

Only include the 5 most architecturally significant decisions (ones that would change the most files if reversed).

### 5.3: Gitignore Suggestion

If `.clarify/` is not in `.gitignore`, suggest (do not modify without asking):
```
Suggest adding .clarify/ to .gitignore — it contains local session data. Add it? [y/N]
```

---

## Phase 6: OFFER

Present completion and next steps:

```
PLAN.md written ({N} decisions, {files_count} files planned).

Start execution? [y/N]
```

Default: no.

If yes:
- If `superpowers:writing-plans` or `superpowers:executing-plans` skills are available, suggest using them
- If `superpowers:using-git-worktrees` is available, suggest creating an isolated worktree
- Otherwise, suggest the user review PLAN.md first and then ask Claude to implement it

If no:
- End the session. The PLAN.md is ready for when they want to proceed.

---

## Language Handling

- Detect the user's language from their initial request
- Conduct the interview in the user's language
- ALL generated files (PLAN.md, session JSON, CLAUDE.local.md entries) remain in **English** regardless of conversation language
- Question axis labels (`[scope]`, `[data]`, etc.) always in English

---

## Error Recovery

| Situation | Action |
|-----------|--------|
| Empty/vague request (no $ARGUMENTS and no context) | Ask: "Describe in one sentence what you want to build" before starting |
| Codebase too large for full scan | Scan top-level structure only, note limitation in PLAN.md |
| PLAN.md write fails | Output plan content directly to the conversation as fallback |
| User abandons mid-interview | Save partial session to `.clarify/sessions/` with `"complete": false` |
| Prior session exists but request is different | Ask: "Found prior session for '{old_request}'. Start fresh? [Y/n]" |
