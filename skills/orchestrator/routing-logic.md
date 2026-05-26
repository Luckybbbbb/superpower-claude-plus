# Routing Logic Reference

Decision tree for selecting execution mode. Read this after completing the clarification phase (Phase 1).

## Decision Tree

```dot
digraph routing {
    rankdir=TB;

    "Task count ≤ 2?" [shape=diamond];
    "Tasks fully independent?" [shape=diamond];
    "Clear stage boundaries?" [shape=diamond];
    "3+ specialized roles?" [shape=diamond];
    "Agent mode" [shape=box, style=filled, fillcolor=lightblue];
    "Workflow mode" [shape=box, style=filled, fillcolor=lightyellow];
    "Team mode" [shape=box, style=filled, fillcolor=lightgreen];
    "Hybrid mode" [shape=box, style=filled, fillcolor=lightcoral];

    "Task count ≤ 2?" -> "Tasks fully independent?" [label="yes"];
    "Task count ≤ 2?" -> "Clear stage boundaries?" [label="no"];
    "Tasks fully independent?" -> "Agent mode" [label="yes"];
    "Tasks fully independent?" -> "Workflow mode" [label="no - ordered deps"];
    "Clear stage boundaries?" -> "3+ specialized roles?" [label="yes"];
    "Clear stage boundaries?" -> "Workflow mode" [label="no"];
    "3+ specialized roles?" -> "Team mode" [label="yes"];
    "3+ specialized roles?" -> "Workflow mode" [label="no"];
}
```

## Mode Selection Criteria

### Agent Mode

**When:** 1-2 independent tasks, no dependencies between them, no specialized roles needed.

**How:**
1. Invoke `superpowers:skill-router` to identify matching skills
2. Build agent prompt with skill injection instructions
3. Dispatch single Agent with the enriched prompt
4. Review result when agent returns

**Example scenario:** "Fix a CSS bug in the header and update the button color"

### Workflow Mode

**When:** 3+ tasks with clear stage boundaries, data flows between stages, or mix of parallel + sequential work.

**How:**
1. Invoke `superpowers:skill-router` for each stage
2. Define phases matching the stages
3. Use `pipeline()` for sequential stages with context flow
4. Use `parallel()` within a stage for independent sub-tasks
5. Each `agent()` call gets skill injection in its prompt

**Key mechanism:** `pipeline()` passes `(prevResult, originalItem, index)` to each stage — upstream output naturally flows downstream.

**Example scenario:** "Design a dashboard UI → Implement components → Write integration tests"

### Team Mode

**When:** 3+ specialized roles that need real-time coordination, complex dependency graph, or tasks need ongoing communication.

**How:**
1. Invoke `superpowers:skill-router` for each role's tasks
2. `TeamCreate` with descriptive name
3. `TaskCreate` for each task, use `addBlockedBy` for dependencies
4. Spawn teammates via `Agent` tool with `team_name` and `name`
5. Assign tasks via `TaskUpdate` with `owner`
6. Teammates communicate via `SendMessage` for context passing
7. Monitor via `TaskList`

**Key mechanism:** `TaskCreate(addBlockedBy)` enforces execution order. `SendMessage` passes context between roles.

**Example scenario:** "Designer creates mockups → Frontend implements → Backend builds API → QA tests integration"

### Hybrid Mode

**When:** Outer orchestration is a pipeline (Workflow), but specific stages need multi-role collaboration (Team).

**How:**
1. Use Workflow for the top-level pipeline
2. For stages needing collaboration, dispatch an agent that internally creates a Team
3. The Team agent acts as mini-lead for that stage

**Example scenario:** "Design system (Team: designer + architect) → Implement (Workflow: parallel components) → Review (Agent: single reviewer)"

## Dependency Graph Construction

For any mode with dependencies:

1. **List all sub-tasks** identified during Phase 1
2. **Identify dependencies** — which tasks need output from other tasks?
3. **Build a DAG:**
   - Tasks with no dependencies → Level 0 (can start immediately)
   - Tasks depending on Level 0 → Level 1
   - Tasks depending on Level 1 → Level 2
   - etc.
4. **Same-level tasks** can run in parallel
5. **Cross-level tasks** must be sequential

This DAG directly maps to:
- Workflow: `parallel()` for same-level, `pipeline()` stages for levels
- Team: `TaskCreate(addBlockedBy)` based on DAG edges

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

```javascript
Agent({
  name: 'designer',
  team_name: 'project-team',
  prompt: 'You are the UI designer.\n\n' + SKILL_INJECTIONS['T2'].join('\n') + '\n\nClaim and complete tasks from TaskList.'
})
```

This contract is enforced by the Phase 4 Pre-flight Checklist in SKILL.md.
