---
name: theoros
description: Invoke when the user runs /theoros or asks to orchestrate, plan, or manage a complex multi-step task with structured discovery, model selection, cost awareness, and visual progress tracking.
version: 1.0.0
argument-hint: [task description]
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep, Agent]
---

# THEOROS — Meta-Orchestrator

Theoros is your meta-workflow layer. It guides complex tasks through structured discovery, model selection, planning, and execution — with token cost awareness and visual progress tracking at every step.

**Named for the ancient Greek concept of careful observation before action.**

---

## Quick Reference: Model Pricing (June 2026)

| Model             | ID                  | Context | Input $/1M | Output $/1M |
|-------------------|---------------------|---------|------------|-------------|
| Claude Fable 5    | `claude-fable-5`    | 1M      | $10.00     | $50.00      |
| Claude Opus 4.8   | `claude-opus-4-8`   | 1M      | $5.00      | $25.00      |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M      | $3.00      | $15.00      |
| Claude Haiku 4.5  | `claude-haiku-4-5`  | 200K    | $1.00      | $5.00       |

---

## Session Persistence

At every phase boundary, write session state to `.theoros/session.json` in the working directory so the session survives `/compact`. Create `.theoros/` if it does not exist.

Session state schema:
```json
{
  "version": "1.0",
  "started_at": "<ISO timestamp>",
  "resumed_at": null,
  "phase": "discover",
  "args": "",
  "problem": {
    "title": "",
    "description": "",
    "scope": "",
    "tech_stack": "",
    "success_criteria": "",
    "constraints": "",
    "urgency": "medium"
  },
  "models": {
    "planning": "",
    "implementation": ""
  },
  "plan": {
    "file": ".theoros/plan.md",
    "steps": [],
    "total": 0,
    "completed": 0
  },
  "tokens": {
    "discovery_est": 0,
    "planning_est": 0,
    "build_est_per_step": 0,
    "build_est_total": 0,
    "overhead": 15000,
    "context_window": 1000000
  },
  "is_repetitive": null,
  "compact_warnings": 0
}
```

---

## ASCII Dashboard

Render this dashboard at every phase transition and after every completed plan step. Replace all bracketed placeholders with actual values.

```
╔══════════════════════════════════════════════════════════════════╗
║  THEOROS v1.0  │  Phase: [PHASE_NAME]    │  [MODEL_SHORT]       ║
╠══════════════════════════════════════════════════════════════════╣
║  WORKFLOW                                                        ║
║  [S1] DISCOVER  →  [S2] PLAN  →  [S3] BUILD  →  [S4] VERIFY     ║
╠══════════════════════════════════════════════════════════════════╣
║  ACTIVE: [CURRENT_TASK — max 52 chars]                          ║
╠══════════════════════════════════════════════════════════════════╣
║  TOKEN BUDGET                                                    ║
║  Discovery:      [DISC_TOKENS] tkn   $[DISC_COST]               ║
║  Planning:      ~[PLAN_TOKENS] tkn   $[PLAN_COST]               ║
║  Build:         ~[BUILD_TOKENS] tkn  $[BUILD_COST]              ║
║  TOTAL EST:     ~[TOTAL_TOKENS] tkn  $[TOTAL_COST]              ║
╠══════════════════════════════════════════════════════════════════╣
║  CONTEXT  [PROGRESS_BAR_16]  [PCT]%  [HEALTH_STATUS]            ║
╚══════════════════════════════════════════════════════════════════╝
```

**Workflow step symbols:**
- `[✓]` — completed
- `[●]` — currently active
- `[ ]` — pending

**Context progress bar (exactly 16 chars):** `█` for used, `░` for free.
- 25% → `████░░░░░░░░░░░░`
- 50% → `████████░░░░░░░░`
- 75% → `████████████░░░░`
- 87% → `██████████████░░`

Formula: `filled = round(pct / 100 * 16)`, bar = `filled × █` + `(16-filled) × ░`

**Context health labels:**
- < 40%  → `✓ Healthy`
- 40–59% → `◉ Good`
- 60–79% → `⚠ Warn — consider /compact`
- ≥ 80%  → `✗ CRITICAL — run /compact now`

---

## Token Estimation Heuristics

Claude Code does not expose a direct token count. Use these estimates:

| Item                              | Tokens        |
|-----------------------------------|---------------|
| System prompt + loaded skills     | ~15,000–25,000|
| Per message/response pair         | ~1,500–3,000  |
| Discovery phase (6–8 Q&A pairs)   | ~12,000–18,000|
| Planning response (Opus)          | ~15,000–30,000|
| Planning response (Sonnet/Haiku)  | ~8,000–18,000 |
| Per build step (Small)            | ~4,000        |
| Per build step (Medium)           | ~8,000        |
| Per build step (Large)            | ~15,000       |

**Context window:**
- Haiku 4.5: 200,000 tokens
- All other current models: 1,000,000 tokens

**Cost formula (assuming 70% input / 30% output split):**
```
cost = (total_tokens × 0.7 × input_rate) + (total_tokens × 0.3 × output_rate)
```
where rates are per-token (divide per-million rate by 1,000,000).

**Context health:**
```
health_pct = estimated_tokens_used / context_window × 100
```

---

## PHASE 0: Initialization

1. **Check for existing session:** Does `.theoros/session.json` exist?
   - If yes: read it and ask:
     > "I found a previous theoros session (started [started_at], phase: [phase], task: [problem.title]). Resume it, or start fresh?"
   - If resume: skip forward to the saved phase, loading all context from the file.
   - If fresh: overwrite the file and continue.

2. **Pre-fill from arguments:** If `$ARGUMENTS` is non-empty, use it as the initial problem description. Skip or shorten questions it already answers in Phase 1.

3. **Create directory and write initial session.json** (phase: "discover").

4. **Display welcome banner:**
```
╔══════════════════════════════════════════════════════════════════╗
║           THEOROS v1.0  —  Meta-Orchestrator                     ║
║  Problem discovery · Model selection · Cost estimation           ║
║  Step-by-step execution · Context health monitoring              ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## PHASE 1: Problem Discovery Interview

Goal: Deeply understand the task before writing a single line of code.

Ask questions **in groups** (not all at once). Adapt or skip questions that `$ARGUMENTS` already answers. Listen carefully to each answer before proceeding.

**Group A — Core Problem**
1. What exactly needs to happen? Describe the desired end state in concrete terms.
2. What triggered this now — deadline, bug report, new feature request, or something else?

**Group B — Scope & Context**
3. Which files, systems, or areas of the codebase are in scope? Anything explicitly out of scope?
4. What tech stack, languages, and frameworks are involved?

**Group C — Success & Constraints**
5. What constraints must be respected — performance, backward compatibility, security, team style guides?
6. How will you know this is done? What does success look like concretely and measurably?

**Group D — Risk & History** *(ask only if complexity is non-trivial)*
7. Have you attempted this before? What happened?
8. Any hidden edge cases or gotchas you already know about?

After all groups, synthesize what you've heard before moving to Phase 2.

**Save all answers to `session.json → problem`.**

---

## PHASE 2: Problem Synthesis & Confirmation

1. Synthesize answers from Phase 1 into a structured problem statement:

```
PROBLEM STATEMENT
─────────────────────────────────────────────────────
Title:             [one-line task name]
Description:       [2–3 sentence summary]
Scope:             [in-scope / out-of-scope]
Tech Stack:        [languages, frameworks, tools]
Success Criteria:  [concrete, measurable definition of done]
Constraints:       [hard limits that cannot be violated]
Urgency:           [low / medium / high / critical]
─────────────────────────────────────────────────────
```

2. Ask: "Does this accurately capture what you need? Any corrections or additions?"

3. Incorporate amendments. Update `session.json → problem`.

4. Render the dashboard with DISCOVER = `[✓]`, all others = `[ ]`.
   Update `session.json → phase: "plan"`.

---

## PHASE 3: Model Selection

1. Display the model selection prompt:

```
MODEL SELECTION — Planning
──────────────────────────────────────────────────────────
Which model should create the plan?

  [1] Claude Opus 4.8    claude-opus-4-8
      Most capable reasoning. Best for ambiguous or complex tasks.
      Cost: $5 input / $25 output per 1M tokens

  [2] Claude Sonnet 4.6  claude-sonnet-4-6
      Balanced speed + quality. Most tasks work well here.
      Cost: $3 input / $15 output per 1M tokens

  [3] Claude Haiku 4.5   claude-haiku-4-5
      Fast and economical. Best for well-defined, simple tasks.
      Cost: $1 input / $5 output per 1M tokens

  [4] Claude Fable 5     claude-fable-5
      Maximum capability. For the most demanding reasoning.
      Cost: $10 input / $50 output per 1M tokens

Your choice (1–4):
──────────────────────────────────────────────────────────
```

2. Wait for selection. Map to model ID:
   - `1` → `claude-opus-4-8`
   - `2` → `claude-sonnet-4-6`
   - `3` → `claude-haiku-4-5`
   - `4` → `claude-fable-5`

3. Save to `session.json → models.planning`.

4. Render dashboard with PLAN = `[●]`.

---

## PHASE 4: Plan Creation

**If planning model is `claude-opus-4-8` or `claude-fable-5`:**

Spawn a dedicated planning agent using the Agent tool:
- `subagent_type`: "Plan"
- `model`: `"opus"` for Opus 4.8, or `"fable"` for Fable 5
- `prompt`: Provide the full problem statement from Phase 2, then instruct:

  > Produce a complete implementation plan with:
  > 1. A numbered list of concrete steps (label each S=small/M=medium/L=large)
  > 2. The 3–5 most critical files or components to touch and why
  > 3. Key risks with mitigations
  > 4. Recommended execution order (note any steps that can run in parallel)
  > Be specific — no hand-waving. Each step should be actionable by itself.

**If planning model is `claude-sonnet-4-6` or `claude-haiku-4-5`:**

Create the plan inline in the current context using the same structure.

**After the plan is ready:**

1. Display it clearly, numbered.
2. Ask: "Does this plan look right? Want to add, remove, or reorder any steps?"
3. Incorporate changes.
4. Write the final plan to `.theoros/plan.md`.
5. Populate `session.json → plan` with steps array, set each `status: "pending"`, set `total`.
6. Estimate planning tokens and update `session.json → tokens.planning_est`.
7. Render dashboard with PLAN = `[✓]`, BUILD = `[ ]`.

---

## PHASE 5: Implementation Model Selection

1. Count plan steps. Classify each as S/M/L (ask if unclear). Estimate build tokens:
   - S step: 4,000 tokens
   - M step: 8,000 tokens
   - L step: 15,000 tokens

2. Display cost comparison:

```
IMPLEMENTATION MODEL
──────────────────────────────────────────────────────────
Plan has [N] steps ([S]×S, [M]×M, [L]×L steps).
Estimated build tokens: ~[BUILD_TOKENS]

Projected cost by model:
  Opus 4.8    ~$[OPUS_COST]   (highest quality)
  Sonnet 4.6  ~$[SONNET_COST] (recommended balance)  ← recommended
  Haiku 4.5   ~$[HAIKU_COST]  (lowest cost)
  Fable 5     ~$[FABLE_COST]  (maximum capability)

Planning was done with: [PLANNING_MODEL]
Which model should handle implementation? (1–4):
──────────────────────────────────────────────────────────
```

3. Wait for selection. Save to `session.json → models.implementation`.

4. Note: Claude Code does not switch models mid-session. Inform the user:
   > "Noted. The active session model will be used for implementation. To use a different model, start a new Claude Code session — `.theoros/session.json` will be picked up automatically on resume."

---

## PHASE 6: Token Budget Dashboard

Before starting execution, display the full projected budget:

1. Sum all estimates: `overhead + discovery + planning + build_total`
2. Calculate projected cost using the implementation model's pricing
3. Calculate context health percentage
4. Render the dashboard with all fields filled in (not placeholder values)

Ask: "Projected total cost: ~$[X.XX] (~[TOTAL_TOKENS] tokens). Ready to execute?"

---

## PHASE 7: Execution

Execute plan steps one at a time.

**For each step:**

1. Print: `━━━ Step [N]/[TOTAL]: [step description] ━━━`
2. Execute the step fully.
3. After completion:
   - Update `session.json → plan.steps[n].status = "done"`
   - Increment `session.json → plan.completed`
   - Add actual tokens used estimate to `session.json → tokens.build_actual`
4. Render the BUILD-phase dashboard:
   - DISCOVER `[✓]`, PLAN `[✓]`, BUILD `[●]` (show `step N/total`), VERIFY `[ ]`
   - Update token totals
   - Update context health

**Context check after every step:** if health ≥ 60%, trigger Phase 8.

---

## PHASE 8: Context Health Monitoring

Trigger based on estimated health percentage.

**At 60–79% (Warning):**

```
┌─────────────────────────────────────────────────────────────────┐
│  ⚠  CONTEXT WARNING                                             │
│                                                                 │
│  Estimated context usage: [XX]%                                 │
│  Consider running /compact before the next step.               │
│                                                                 │
│  Session is saved → .theoros/session.json                       │
│  After /compact, run /theoros to resume from step [N+1].       │
└─────────────────────────────────────────────────────────────────┘
```

Increment `session.json → compact_warnings`.

Ask: "Run /compact now or continue to the next step?"

**At ≥ 80% (Critical):**

```
┌─────────────────────────────────────────────────────────────────┐
│  ✗  CONTEXT CRITICAL — ACTION REQUIRED                          │
│                                                                 │
│  Estimated context usage: [XX]%. Risk of truncation.           │
│                                                                 │
│  Steps done: [N]/[TOTAL]                                        │
│  1. Run /compact now                                            │
│  2. Then run /theoros — it will auto-resume from step [N+1]     │
└─────────────────────────────────────────────────────────────────┘
```

Pause execution. Do not continue until the user acknowledges.

**On resume after /compact:**
- Read `.theoros/session.json`
- Say: "Resuming theoros. [N]/[TOTAL] steps complete. Continuing from step [N+1]: [description]..."
- Continue from the next pending step

---

## PHASE 9: Completion & Retrospective

After all plan steps have `status: "done"`:

1. Run relevant verification based on task type:
   - Code changes: run tests, linter, type checker via Bash
   - Config changes: validate syntax
   - Database changes: verify migration applied correctly

2. Render the final dashboard:
   - All phases `[✓]`, VERIFY `[✓]`
   - Final token totals (sum of all estimates)
   - Context health (informational only — task is done)

3. Show completion summary:

```
TASK COMPLETE
──────────────────────────────────────────
Title:        [problem.title]
Steps done:   [N]/[N]
Est. tokens:  ~[TOTAL_TOKENS]
Est. cost:    ~$[TOTAL_COST]
Model (plan): [PLANNING_MODEL]
Model (impl): [IMPL_MODEL]
──────────────────────────────────────────
```

4. Ask the retrospective question:

```
RETROSPECTIVE
─────────────────────────────────────────────────────
Is this a task you'll need to do regularly?

  [y] Yes — let's create a reusable skill for it
  [n] No  — one-off, we're done

```

5. **If yes (repetitive):**

   Ask: "Describe the repeatable pattern in one sentence — what's the constant, and what varies each time?"

   Offer skill creation options:

   ```
   SKILL CREATION
   ──────────────────────────────────────────────────────
   I can turn this into a /[skill-name] command. Where should it live?

     [1] This project only — .claude/skills/[name]/SKILL.md
     [2] Your theoros plugin — skills/[name]/SKILL.md
     [3] Show me the template — I'll write it myself

   ```

   For options 1 or 2: draft a SKILL.md based on the task pattern (see Skill Template reference below).

6. Write final `session.json` with `phase: "done"`.

---

## Reference: Skill Template

Use this when creating a new skill in Phase 9:

```markdown
---
name: [skill-name-kebab-case]
description: [One sentence: when to invoke this skill and what it does]
version: 1.0.0
argument-hint: [required-arg] [optional-arg]
allowed-tools: [Read, Write, Edit, Bash, Glob, Grep]
---

# [Skill Name]

[Brief description — what problem this solves]

## Usage

/[skill-name] [required-arg] [optional-arg]

## Steps

1. [Step 1]
2. [Step 2]
3. [Step 3]

## Reference

[Any tables, code snippets, or config this skill needs to reference]
```

---

## Reference: Dashboard Rendering Rules

- Box width: 68 characters inside `║` delimiters
- Pad every line with trailing spaces to fill the full width before the closing `║`
- Workflow line shows exactly 4 phases separated by ` → `
- Token numbers: use commas for thousands (e.g., `14,500`)
- Cost: always show 2 decimal places (e.g., `$0.43`)
- Never render the dashboard with unfilled `[PLACEHOLDER]` values — compute them first
