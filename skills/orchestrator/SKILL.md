---
name: orchestrator
description: Use when a task is too complex for a single agent dispatch — multiple sub-tasks that span different domains like UI and backend, tasks with dependencies or ordering requirements, or requests involving parallel work that needs coordination. Also triggers when task mentions pipeline, workflow, team collaboration, or multi-step implementation.
---

# Orchestrator

## Overview

Intelligent task orchestrator that analyzes complex requests, maps them to specialized skills, selects the optimal execution mode (Agent / Workflow / Team), and coordinates execution with dependency management and context passing.

**Core principle:** Clarify first, route skills, then execute with the right tool for the job.

**Announce at start:** "I'm using the orchestrator skill to coordinate this task."

## When to Use

```dot
digraph when_to_use {
    "Complex task received?" [shape=diamond];
    "Has approved spec?" [shape=diamond];
    "Multiple sub-tasks?" [shape=diamond];
    "use superpowers:orchestrator" [shape=box, style=filled, fillcolor=lightgreen];
    "Use single skill or agent" [shape=box];
    "Use superpowers:brainstorming first" [shape=box];

    "Complex task received?" -> "Has approved spec?" [label="yes"];
    "Has approved spec?" -> "use superpowers:orchestrator" [label="yes - from brainstorming"];
    "Has approved spec?" -> "Multiple sub-tasks?" [label="no"];
    "Multiple sub-tasks?" -> "use superpowers:orchestrator" [label="yes"];
    "Multiple sub-tasks?" -> "Use single skill or agent" [label="no"];
}
```

**Two entry paths:**
- **From brainstorming** — Spec already written and approved. Read spec, skip clarification, go directly to skill routing.
- **Direct invocation** — No spec exists. User describes a coordination task (parallel bugs, multi-step refactor, etc.). Do quick clarification first.

**Use when:**
- Task spans multiple domains (UI + backend, design + implementation)
- Multiple sub-tasks with dependencies between them
- Need to run tasks in parallel with context flow
- Task complexity is beyond simple sequential agent dispatch
- Brainstorming completed and spec is ready for execution

**Don't use when:**
- Simple single-domain task → use the relevant skill directly
- Need to design something new → use `superpowers:brainstorming` first
- Already have a written implementation plan → use `superpowers:subagent-driven-development`

## The Four Phases

Phases execute in order: **Clarify → Route Skills → Decide Mode → Execute**

### Phase 1: Understand the Task

**Two modes based on entry path:**

**Mode A — Spec Available (from brainstorming):**
1. Read the spec document
2. Extract sub-tasks, dependencies, and technical domains from the spec
3. Skip clarification — spec already captures user intent
4. Proceed directly to Phase 2

**Mode B — No Spec (direct invocation):**
1. **Explore context** — Check current project state (files, recent changes, relevant code)
2. **Ask one question at a time** — Understand: overall goal, sub-tasks, dependencies, domains, constraints
3. **Confirm understanding** — Restate the task in your own words, ask for approval

**Scope check:** If the task requires open-ended design exploration (not just coordination), stop and recommend `superpowers:brainstorming` first. Orchestrator coordinates known work — it does not design from scratch.

**Decision: Is this too large for one plan?** If the spec or task description covers multiple independent subsystems, flag this. Suggest decomposing into separate spec → orchestrator cycles, one per subsystem.

**Output of Phase 1:** A structured list of sub-tasks with dependencies noted.

### Phase 2: Route Skills

Invoke `superpowers:skill-router` (via Skill tool) to identify which skills should be loaded for each sub-task.

**Process:**

1. Invoke Skill tool with `skill="superpowers:skill-router"`, passing the sub-task list
2. Review the matched skills and their injection instructions
3. Note which skills apply to which sub-tasks

**If no skills match:** That's fine — some tasks don't need specialized skills. Proceed without injection.

### Phase 3: Decide Execution Mode

Read the routing logic in `routing-logic.md` (in this skill's directory) and select the execution mode.

**Quick decision:**

| Signal | Mode |
|--------|------|
| 1-2 tasks, no dependencies | **Agent** |
| 3+ tasks with stage flow | **Workflow** |
| 3+ specialized roles, real-time comms | **Team** |
| Pipeline + multi-role stages | **Hybrid** |

### Phase 4: Execute

**PRE-FLIGHT CHECKPOINT (MANDATORY):**

Before writing ANY Workflow script, Team config, or Agent prompt, complete this checklist:

1. Phase 2 produced skill routing results (even if "no matches found")
2. For each sub-task with matched skills, prepare the injection string:
   `"IMPORTANT: Before starting, invoke the Skill tool with skill=\"{matched-skill-name}\" to load domain-specific guidance."`
3. Build a `SKILL_INJECTIONS` map at the top of your Workflow script (see routing-logic.md "Phase 2 → Phase 4 Data Contract")
4. Verify EVERY `agent()` call includes skill injection when SKILL_INJECTIONS has entries for that task

If any `agent()` prompt is missing skill injection when Phase 2 found matches, STOP and fix before continuing.

Route based on execution mode decided in Phase 3.

**Workflow Mode — Direct Execution:**
Build a Workflow script using patterns from `workflow-patterns.md`. Do NOT invoke writing-plans — Workflow handles its own task construction. Each `agent()` call gets skill injection. Use `pipeline()` for sequential stages, `parallel()` for independent tasks.

**Team Mode — Direct Execution:**
Create team, create tasks with `addBlockedBy` dependencies, spawn teammates with skill injection. Do NOT invoke writing-plans — Team handles its own coordination. Monitor via `TaskList`, shutdown when done.

**Agent Mode — Via writing-plans:**
If sub-tasks need detailed implementation plans (code steps, file paths, test commands), invoke `superpowers:writing-plans`. Pass an **execution context** that writing-plans embeds in the plan header:

```
Execution Context (pass to writing-plans):
- execution_mode: agent
- skill_injections: {task-name: ["skill-a", "skill-b"]}
- dependencies: {task-3: [task-1, task-2]}
- parallel_groups: [[task-1, task-2]]
```

If sub-tasks are simple enough (clear scope, 1-2 files each), skip writing-plans and dispatch agents directly.

**Skill injection pattern (all modes):** Append to every agent prompt: `Before starting, invoke the Skill tool with skill="{matched-skill-name}" to load domain-specific guidance.`

## Context Passing Between Tasks

**Pipeline:** `pipeline()` passes `(prevResult, originalItem, index)` — upstream output flows downstream.
**Team:** `SendMessage` passes context between roles. Downstream reads from messages and `TaskGet`.
**Agent:** No context passing — single task.

## Handling Problems During Execution

- **Agent BLOCKED:** Re-dispatch with more context or break task down
- **Unexpected results:** Pipeline continues with what's available
- **Team task stalled:** Check `TaskList`, message the owner, re-assign if needed
- **No skills matched:** Proceed without injection — not every task needs specialized skills

## Red Flags

| Thought | Reality |
|---------|---------|
| "I can just dispatch agents sequentially" | Sequential dispatch loses context between tasks. Use the orchestrator. |
| "This is too complex to orchestrate" | That's exactly when orchestration is most valuable. Break it down further in Phase 1. |
| "I'll skip the clarification phase" | Unclear requirements → wrong orchestration → wasted work. Always clarify first. |
| "I don't need skill routing" | Skill injection ensures domain expertise is loaded. Never skip Phase 2. |
| "One mode fits all" | Wrong mode = wrong coordination. Use the routing logic. |

## Integration

**Required workflow skills:**
- **superpowers:brainstorming** — Design phase (precedes orchestrator for new features). Terminal state of brainstorming is invoking orchestrator.
- **superpowers:skill-router** — Phase 2 skill discovery
- **superpowers:writing-plans** — Creates detailed implementation plan (invoked by orchestrator when spec needs task breakdown)
- **superpowers:dispatching-parallel-agents** — Simplified parallel dispatch (alternative to full Workflow for simple cases)
- **superpowers:subagent-driven-development** — Sequential plan execution (alternative for pre-written plans)

**Supporting references (in this skill's directory):**
- `routing-logic.md` — Decision tree for mode selection
- `workflow-patterns.md` — Construction patterns for Workflow scripts and Team setup
