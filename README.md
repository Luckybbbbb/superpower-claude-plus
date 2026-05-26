# Superpower Claude Plus

Enhanced superpowers plugin for Claude Code with orchestrator coordination, skill-router auto-discovery, and improved Team/Workflow integration. Forked from [obra/superpowers](https://github.com/obra/superpowers).

## What's Different from Upstream

- **Orchestrator skill** — Coordinates complex multi-domain tasks with automatic execution mode selection (Agent/Workflow/Team)
- **Skill-router** — Zero-config semantic scanning that auto-discovers and injects relevant local skills into agent prompts
- **Hooks enforcement** — PostToolUse and PreToolUse hooks that reinforce the brainstorming → orchestrator → writing-plans flow
- **Execution strategy passthrough** — Orchestrator passes parallel groups, dependencies, and skill injections to writing-plans
- **Dual-entry orchestrator** — Receives tasks from brainstorming AND can be invoked directly for coordination-only tasks

## Quickstart

Give your agent Superpowers: [Claude Code](#claude-code), [Codex CLI](#codex-cli), [Cursor](#cursor).

## How it works

It starts from the moment you fire up your coding agent. As soon as it sees that you're building something, it *doesn't* just jump into trying to write code. Instead, it steps back and asks you what you're really trying to do.

Once it's teased a spec out of the conversation, it shows it to you in chunks short enough to actually read and digest.

After you've signed off on the design, the **orchestrator** kicks in. It analyzes the spec, discovers which local skills are relevant (via skill-router), selects the optimal execution mode, and coordinates implementation. For complex multi-domain tasks, it uses Claude's native Team or Workflow features instead of serial subagent dispatch.

Next up, once the orchestrator has a plan, it launches agents — potentially in parallel — to work through each engineering task. The skill-router ensures each agent gets the right domain expertise loaded automatically.

There's a bunch more to it, but that's the core of the system. And because the skills trigger automatically, you don't need to do anything special. Your coding agent just has Superpowers.

## Installation

### Claude Code

#### From this repository (recommended)

Register this marketplace and install:

```bash
/plugin marketplace add Luckybbbbb/superpower-claude-plus
```

```bash
/plugin install superpower-claude-plus@superpower-claude-plus
```

#### Official Marketplace (upstream)

If you prefer the upstream version without orchestrator enhancements:

```bash
/plugin install superpowers@claude-plugins-official
```

### Codex CLI

```bash
droid plugin marketplace add https://github.com/Luckybbbbb/superpower-claude-plus
```

```bash
droid plugin install superpower-claude-plus@superpower-claude-plus
```

### Gemini CLI

```bash
gemini extensions install https://github.com/Luckybbbbb/superpower-claude-plus
```

### OpenCode

```
Fetch and follow instructions from https://raw.githubusercontent.com/Luckybbbbb/superpower-claude-plus/refs/heads/main/.opencode/INSTALL.md
```

### Cursor

- In Cursor Agent chat, install from marketplace:

  ```text
  /add-plugin superpower-claude-plus
  ```

## The Workflow

1. **brainstorming** - Activates before writing code. Refines rough ideas through questions, explores alternatives, presents design in sections for validation. Saves design document.

2. **orchestrator** - Activates after brainstorming. Analyzes spec, routes relevant skills via skill-router, selects execution mode (Agent/Workflow/Team), coordinates implementation.

3. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

4. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps. Receives execution strategy from orchestrator.

5. **subagent-driven-development** or **executing-plans** - Activates with plan. Dispatches fresh subagent per task with two-stage review (spec compliance, then code quality), or executes in batches with human checkpoints.

6. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

7. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

8. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## What's Inside

### Skills Library

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR cycle (includes testing anti-patterns reference)

**Debugging**
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed

**Collaboration**
- **brainstorming** - Socratic design refinement
- **orchestrator** - Multi-domain task coordination with skill-router integration
- **skill-router** - Zero-config semantic scanning for skill auto-discovery
- **writing-plans** - Detailed implementation plans with execution strategy
- **executing-plans** - Batch execution with checkpoints
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow
- **subagent-driven-development** - Fast iteration with two-stage review (spec compliance, then code quality)

**Meta**
- **writing-skills** - Create new skills following best practices (includes testing methodology)
- **using-superpowers** - Introduction to the skills system

## Hooks

This plugin includes three hooks:

| Event | Purpose |
|-------|---------|
| **SessionStart** | Injects `using-superpowers` skill into every session |
| **PostToolUse (Skill)** | After brainstorming loads, reminds that orchestrator is the next step |
| **PreToolUse (Skill)** | When writing-plans is invoked, reminds to check execution strategy |

## Philosophy

- **Test-Driven Development** - Write tests first, always
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success
- **Smart coordination** - Use the right execution mode for each task

## Updating

Plugin updates are automatic when using the marketplace. Re-run the install command to update manually.

## License

MIT License - see LICENSE file for details

## Credits

Forked from [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra).
