# Orchestrator Skill Injection Pipeline Fix

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 5 discrepancies between orchestrator/skill-router design intent and actual LLM execution behavior, ensuring skill injection results from Phase 2 are always consumed by Phase 4.

**Architecture:** All changes are to SKILL.md documentation files (LLM instructions). The fix introduces HARD-GATE checkpoints and structured data contracts to prevent the LLM from skipping the skill injection step when transitioning from Phase 2 (routing) to Phase 4 (execution).

**Tech Stack:** Markdown skill documents, no code changes.

**Execution Strategy:** Sequential execution

- Mode: sequential
- Parallel groups: none
- Dependencies: Task 3 depends on Task 2 (routing-logic data contract must exist before orchestrator references it)
- Skill injections: none

---

### Task 1: Add Phase 4 Pre-flight Checklist to orchestrator SKILL.md

**Files:**
- Modify: `skills/orchestrator/SKILL.md:100-124` (Phase 4 section)

**Dependencies:** None.

- [ ] **Step 1: Add pre-flight checklist before Phase 4 execution instructions**

In `skills/orchestrator/SKILL.md`, insert the following block **after** line 100 (`### Phase 4: Execute`) and **before** the existing "Route based on execution mode" content (line 103):

```markdown
**PRE-FLIGHT CHECKPOINT (MANDATORY):**

Before writing ANY Workflow script, Team config, or Agent prompt, complete this checklist:

1. Phase 2 produced skill routing results (even if "no matches found")
2. For each sub-task with matched skills, prepare the injection string:
   `"IMPORTANT: Before starting, invoke the Skill tool with skill=\"{matched-skill-name}\" to load domain-specific guidance."`
3. Build a `SKILL_INJECTIONS` map at the top of your Workflow script (see routing-logic.md "Phase 2 → Phase 4 Data Contract")
4. Verify EVERY `agent()` call includes skill injection when SKILL_INJECTIONS has entries for that task

If any `agent()` prompt is missing skill injection when Phase 2 found matches, STOP and fix before continuing.
```

- [ ] **Step 2: Verify the edit is correctly placed**

Read `skills/orchestrator/SKILL.md` and confirm:
- The pre-flight checkpoint appears between the `### Phase 4: Execute` heading and the "Route based on execution mode" subsections
- The existing content (Workflow Mode, Team Mode, Agent Mode descriptions) is preserved below
- Line 124's existing skill injection pattern description remains unchanged as reinforcement

- [ ] **Step 3: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/orchestrator/SKILL.md
git commit -m "fix(orchestrator): add Phase 4 pre-flight checkpoint for skill injection"
```

---

### Task 2: Add Phase 2 → Phase 4 Data Contract to routing-logic.md

**Files:**
- Modify: `skills/orchestrator/routing-logic.md` (append at end)

**Dependencies:** None.

- [ ] **Step 1: Append data contract section to routing-logic.md**

Append the following section at the end of `skills/orchestrator/routing-logic.md` (after the "Dependency Graph Construction" section):

```markdown

## Phase 2 → Phase 4 Data Contract

Phase 2 (skill-router) output MUST be materialized as a `SKILL_INJECTIONS` object at the top of any Workflow script or Team config. This ensures the routing results survive the transition from Phase 2 analysis to Phase 4 execution.

**Workflow scripts — add at the top of the script body:**

```javascript
// === SKILL INJECTIONS (from Phase 2 routing) ===
// MUST be filled before writing any agent() call
const SKILL_INJECTIONS = {
  'T1': [],  // no skills matched
  'T2': ['IMPORTANT: Before starting, invoke the Skill tool with skill="ui-ux-pro-max:ui-ux-pro-max" to load domain-specific guidance.'],
  // ... fill for each task
}
```

**Using injections in agent() calls:**

```javascript
// For tasks with matched skills — append injection to prompt
await agent(
  'Implement responsive layout...\n\n' + SKILL_INJECTIONS['T2'].join('\n'),
  { label: 'T2', phase: 'Core Components' }
)

// For tasks with no matches — no injection needed
await agent(
  'Install dependencies...',
  { label: 'T1', phase: 'Foundation' }
)
```

**Team mode — include in each teammate's prompt:**

```
Agent({
  name: 'designer',
  team_name: 'project-team',
  prompt: 'You are the UI designer.\n\n' + SKILL_INJECTIONS['T2'].join('\n') + '\n\nClaim and complete tasks from TaskList.'
})
```

This contract is enforced by the Phase 4 Pre-flight Checklist in SKILL.md.
```

- [ ] **Step 2: Verify the append is correct**

Read `skills/orchestrator/routing-logic.md` and confirm:
- The new section appears after the existing "Dependency Graph Construction" section
- No existing content was modified
- The code blocks are properly formatted

- [ ] **Step 3: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/orchestrator/routing-logic.md
git commit -m "fix(orchestrator): add Phase 2→4 data contract with SKILL_INJECTIONS pattern"
```

---

### Task 3: Enforce SKILL_INJECTIONS pattern in orchestrator SKILL.md Phase 4

**Files:**
- Modify: `skills/orchestrator/SKILL.md:106-124` (Phase 4 Workflow Mode and Skill injection pattern sections)

**Dependencies:** Task 2 must be completed (routing-logic.md data contract must exist).

- [ ] **Step 1: Update Workflow Mode description to reference the data contract**

In `skills/orchestrator/SKILL.md`, replace the existing "Workflow Mode — Direct Execution:" bullet (line 106) with:

```markdown
**Workflow Mode — Direct Execution:**
Build a Workflow script using patterns from `workflow-patterns.md`. Do NOT invoke writing-plans — Workflow handles its own task construction.

**CRITICAL:** Include a `SKILL_INJECTIONS` map at the top of your script (see `routing-logic.md` "Phase 2 → Phase 4 Data Contract"). Every `agent()` call MUST append the injection string from this map for tasks with matched skills. Use `pipeline()` for sequential stages, `parallel()` for independent tasks.
```

- [ ] **Step 2: Update Team Mode description similarly**

In `skills/orchestrator/SKILL.md`, replace the existing "Team Mode — Direct Execution:" bullet (line 109) with:

```markdown
**Team Mode — Direct Execution:**
Create team, create tasks with `addBlockedBy` dependencies, spawn teammates with skill injection. Do NOT invoke writing-plans — Team handles its own coordination. Monitor via `TaskList`, shutdown when done.

**CRITICAL:** Include skill injection strings in each teammate's prompt. See `routing-logic.md` "Phase 2 → Phase 4 Data Contract" for the pattern.
```

- [ ] **Step 3: Verify edits**

Read `skills/orchestrator/SKILL.md` and confirm:
- Workflow Mode and Team Mode both reference the `SKILL_INJECTIONS` data contract
- The existing "Skill injection pattern" line (original line 124) is preserved as additional reinforcement
- No duplicate content

- [ ] **Step 4: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/orchestrator/SKILL.md
git commit -m "fix(orchestrator): enforce SKILL_INJECTIONS in Workflow and Team mode descriptions"
```

---

### Task 4: Add Placeholder Checklist to workflow-patterns.md

**Files:**
- Modify: `skills/orchestrator/workflow-patterns.md` (insert before Pattern 1)

**Dependencies:** None.

- [ ] **Step 1: Insert placeholder validation section before Pattern 1**

In `skills/orchestrator/workflow-patterns.md`, insert the following at the **very beginning** of the file content (after the title line "# Workflow & Team Patterns Reference"), before "## Pattern 1: Simple Pipeline":

```markdown

## Before Using Any Pattern — Required Placeholders

Every `agent()` call in your script MUST include these elements:

1. **Skill injection** — Append the injection string from `SKILL_INJECTIONS[taskName]`. If the array is empty, no injection needed.
2. **Upstream context** — For pipeline stages, prepend `"Previous stage output:\n" + JSON.stringify(prevResult)`. Not needed for Level 0 tasks.
3. **Task description** — The actual implementation instructions.

**Validation:** After writing your script, search for every `agent(` call and verify each one has:
- Either a `SKILL_INJECTIONS[...]` reference or a comment explaining "no skills matched"
- Either upstream context injection or a comment "Level 0 — no upstream dependency"

If any `agent()` call is missing these, fix before proceeding.
```

- [ ] **Step 2: Verify insertion**

Read `skills/orchestrator/workflow-patterns.md` and confirm:
- The new section appears between the file title and "## Pattern 1: Simple Pipeline"
- All 4 existing patterns are preserved unchanged

- [ ] **Step 3: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/orchestrator/workflow-patterns.md
git commit -m "fix(orchestrator): add required placeholder checklist to workflow patterns"
```

---

### Task 5: Enforce Structured Output in skill-router SKILL.md

**Files:**
- Modify: `skills/skill-router/SKILL.md` (replace Output Format section and add HARD-GATE)

**Dependencies:** None.

- [ ] **Step 1: Replace the Output Format section**

In `skills/skill-router/SKILL.md`, replace the entire "## Output Format" section (from "## Output Format" through the closing triple-backtick code block, ending at "```") with:

```markdown
## Output Format (MANDATORY)

<HARD-GATE>
Your output MUST follow this exact structured format. The orchestrator's Phase 4 consumes this output programmatically. Do NOT summarize in prose, tables, or narrative. Do NOT omit the INJECT lines. Each task MUST have its injection strings ready to copy-paste into agent prompts.
</HARD-GATE>

```
=== SKILL ROUTING RESULTS ===

TASK: {task-id-or-description}
STATUS: {MATCHED | NO_MATCH}
  1. {skill-name} — {match reason: which keywords overlapped}
     INJECT: "IMPORTANT: Before starting, invoke the Skill tool with skill=\"{skill-name}\" to load domain-specific guidance."
  2. {skill-name} — {match reason}
     INJECT: "IMPORTANT: Before starting, invoke the Skill tool with skill=\"{skill-name}\" to load domain-specific guidance."

TASK: {next-task-id-or-description}
STATUS: NO_MATCH
(no specialized skills needed for this task)

=== END ROUTING RESULTS ===
```

Every TASK block MUST include STATUS and either INJECT lines or a NO_MATCH explanation.
```

- [ ] **Step 2: Verify the replacement**

Read `skills/skill-router/SKILL.md` and confirm:
- The old prose-based Output Format is completely replaced
- The HARD-GATE block is present
- The "Common Mistakes" and "Real-World Matching Examples" sections that follow are preserved
- The code fence formatting is correct

- [ ] **Step 3: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/skill-router/SKILL.md
git commit -m "fix(skill-router): enforce structured output format with HARD-GATE"
```

---

### Task 6: Add Git Commit HARD-GATE to brainstorming SKILL.md

**Files:**
- Modify: `skills/brainstorming/SKILL.md:109-114` (Documentation section)

**Dependencies:** None.

- [ ] **Step 1: Replace the Documentation section with HARD-GATE version**

In `skills/brainstorming/SKILL.md`, replace the "Documentation:" subsection (lines 109-114, starting with `**Documentation:**` through `- Commit the design document to git`) with:

```markdown
**Documentation:**

- Write the validated design (spec) to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
  - (User preferences for spec location override this default)
- Use elements-of-style:writing-clearly-and-concisely skill if available

<HARD-GATE>
You MUST commit the spec to git immediately after writing it. Run `git add docs/superpowers/specs/... && git commit -m "docs: add design spec for ..."` before proceeding to Spec Self-Review. Do NOT skip this commit.
</HARD-GATE>
```

- [ ] **Step 2: Verify the edit**

Read `skills/brainstorming/SKILL.md` and confirm:
- The HARD-GATE replaces the previous plain `- Commit the design document to git` line
- The "Spec Self-Review" section that follows is preserved
- The overall document flow is logical: Write spec → COMMIT → Self-review → User review

- [ ] **Step 3: Commit**

```bash
cd E:/AIDemos/superpower-claude-plus
git add skills/brainstorming/SKILL.md
git commit -m "fix(brainstorming): add HARD-GATE for spec git commit"
```

---

## Self-Review

**1. Spec coverage check:**

| Discrepancy from analysis | Plan task covering it |
|---------------------------|----------------------|
| #1 Skill injection missing in Phase 4 | Task 1 (pre-flight checklist) + Task 3 (enforce in mode descriptions) |
| #2 skill-router output format non-standard | Task 5 (HARD-GATE structured output) |
| #3 Phase 2→4 data passing broken | Task 2 (SKILL_INJECTIONS data contract) |
| #4 workflow patterns not referenced | Task 4 (placeholder checklist) |
| #5 spec not committed to git | Task 6 (HARD-GATE git commit) |

All 5 discrepancies are covered.

**2. Placeholder scan:** No TBD, TODO, or vague instructions. All steps contain exact content to insert/replace.

**3. Type consistency:** All references to `SKILL_INJECTIONS` object use consistent format across Task 2 (definition), Task 3 (references), and Task 4 (validation).
