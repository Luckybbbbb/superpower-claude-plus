# Workflow & Team Patterns Reference

## Before Using Any Pattern — Required Placeholders

Every `agent()` call in your script MUST include these elements:

1. **Skill injection** — Append the injection string from `SKILL_INJECTIONS[taskName]`. If the array is empty, no injection needed.
2. **Upstream context** — For pipeline stages, prepend `"Previous stage output:\n" + JSON.stringify(prevResult)`. Not needed for Level 0 tasks.
3. **Task description** — The actual implementation instructions.

**Validation:** After writing your script, search for every `agent(` call and verify each one has:
- Either a `SKILL_INJECTIONS[...]` reference or a comment explaining "no skills matched"
- Either upstream context injection or a comment "Level 0 — no upstream dependency"

If any `agent()` call is missing these, fix before proceeding.

Patterns for building Workflow scripts and Team configurations. Use these as templates when constructing the execution phase.

## Pattern 1: Simple Pipeline

For tasks that flow through sequential stages with context passing.

```javascript
export const meta = {
  name: 'pipeline-{task-name}',
  description: '{task description}',
  phases: [
    { title: 'Stage 1 Name' },
    { title: 'Stage 2 Name' },
    { title: 'Stage 3 Name' }
  ]
}

// Dynamically constructed based on DAG from routing-logic.md
const tasks = [
  { name: 'task-1', prompt: '...', skills: ['skill-a'] },
  { name: 'task-2', prompt: '...', skills: ['skill-b'], dependsOn: ['task-1'] },
  { name: 'task-3', prompt: '...', skills: ['skill-c'], dependsOn: ['task-2'] }
]

phase('Stage 1 Name')
const result1 = await agent(tasks[0].prompt + '\n\nIMPORTANT: Before starting, invoke Skill tool with skill="' + tasks[0].skills[0] + '"', {label: tasks[0].name, schema: RESULT_SCHEMA})

phase('Stage 2 Name')
const result2 = await agent('Previous stage output:\n' + JSON.stringify(result1) + '\n\n' + tasks[1].prompt + '\n\nIMPORTANT: Before starting, invoke Skill tool with skill="' + tasks[1].skills[0] + '"', {label: tasks[1].name, schema: RESULT_SCHEMA})

phase('Stage 3 Name')
const result3 = await agent('Previous stage output:\n' + JSON.stringify(result2) + '\n\n' + tasks[2].prompt + '\n\nIMPORTANT: Before starting, invoke Skill tool with skill="' + tasks[2].skills[0] + '"', {label: tasks[2].name, schema: RESULT_SCHEMA})

return { result1, result2, result3 }
```

## Pattern 2: Pipeline with Parallel Branches

For stages where some tasks can run concurrently.

```javascript
export const meta = {
  name: 'pipeline-parallel-{task-name}',
  description: '{task description}',
  phases: [
    { title: 'Design' },
    { title: 'Implement' },
    { title: 'Integrate' }
  ]
}

const designResult = { /* from Phase 1 */ }

phase('Implement')
const implResults = await parallel([
  () => agent('Implement component A based on design:\n' + JSON.stringify(designResult) + '\n\n{skill-injection}', {label: 'impl-a', phase: 'Implement', schema: RESULT_SCHEMA}),
  () => agent('Implement component B based on design:\n' + JSON.stringify(designResult) + '\n\n{skill-injection}', {label: 'impl-b', phase: 'Implement', schema: RESULT_SCHEMA})
])

phase('Integrate')
const integrated = await agent('Integrate these components:\n' + JSON.stringify(implResults) + '\n\n{task-description}', {label: 'integrate', schema: RESULT_SCHEMA})

return { designResult, implResults, integrated }
```

## Pattern 3: Full Pipeline with auto-parallel Detection

Use `pipeline()` to let each item flow through stages independently. Items progress at their own pace — no barrier between stages.

```javascript
export const meta = {
  name: 'auto-pipeline-{task-name}',
  description: '{task description}',
  phases: [
    { title: 'Analyze' },
    { title: 'Implement' },
    { title: 'Verify' }
  ]
}

const items = [/* sub-tasks from Phase 1 */]

const results = await pipeline(
  items,
  async (item) => {
    phase('Analyze')
    return await agent(`Analyze: ${item.description}\n\n{skill-injection}`, {label: `analyze:${item.name}`, phase: 'Analyze', schema: ANALYSIS_SCHEMA})
  },
  async (analysis, item) => {
    phase('Implement')
    return await agent(`Analysis:\n${JSON.stringify(analysis)}\n\nImplement: ${item.description}\n\n{skill-injection}`, {label: `impl:${item.name}`, phase: 'Implement', schema: IMPL_SCHEMA})
  },
  async (impl, item) => {
    phase('Verify')
    return await agent(`Implementation:\n${JSON.stringify(impl)}\n\nVerify: ${item.description}`, {label: `verify:${item.name}`, phase: 'Verify', schema: VERIFY_SCHEMA})
  }
)

return results
```

## Pattern 4: Team Collaboration

For multi-role tasks requiring ongoing communication.

**Construction steps (not a script — Team mode uses tool calls):**

1. **Create team:** `TeamCreate({ team_name: 'task-name-team', description: '...' })`

2. **Create tasks with dependencies:**
```
TaskCreate({ subject: 'Design UI', description: '...', addBlocks: ['Implement UI', 'QA Test'] })
TaskCreate({ subject: 'Implement UI', description: '...', addBlockedBy: ['Design UI'], addBlocks: ['QA Test'] })
TaskCreate({ subject: 'QA Test', description: '...', addBlockedBy: ['Implement UI'] })
```

3. **Spawn teammates:**
```
Agent({ name: 'designer', team_name: 'task-name-team', prompt: 'You are the UI designer. Use Skill tool to load ui-ux-pro-max:ui-ux-pro-max before designing. Claim and complete design tasks from TaskList.' })
Agent({ name: 'implementer', team_name: 'task-name-team', prompt: 'You are the frontend implementer. Claim and complete implementation tasks from TaskList.' })
Agent({ name: 'reviewer', team_name: 'task-name-team', prompt: 'You are the QA reviewer. Claim and complete review tasks from TaskList.' })
```

4. **Monitor:** Check `TaskList` periodically. When a teammate completes, it sends you a message automatically.

5. **Shutdown:** When all tasks complete, `SendMessage({ to: 'designer', message: { type: 'shutdown_request', reason: 'All tasks complete' } })` for each teammate. Then `TeamDelete()`.

## Context Passing Patterns

### Pipeline prevResult
```
Stage 1 output → Stage 2 input (automatic via pipeline callback)
```

### Team SendMessage
```
designer completes → SendMessage({ to: 'implementer', message: 'Design complete. Key decisions: ...' })
```

### Agent prompt injection
```
Agent prompt includes: "Upstream context: {JSON.stringify(previousResult)}"
```
