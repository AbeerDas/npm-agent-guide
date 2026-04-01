---
tags: [planning, reasoning, task-decomposition, todos]
complexity: intermediate
prerequisites: [00_foundations, 01_agent_loop]
---

# Planning & Reasoning

> How to implement plan modes, task decomposition, todo systems, and structured reasoning that make agents more reliable on complex tasks.

---

## Pattern (Universal)

Complex tasks fail when agents jump straight to execution. Planning patterns force agents to think before acting — decomposing tasks, considering alternatives, and creating structured execution plans. This dramatically improves success rates on multi-step tasks.

---

## Plan Mode Architecture

Plan mode is a restricted execution mode where the agent can research and plan but not make changes. This separates "thinking" from "doing."

### How It Works

```
Normal Mode                          Plan Mode
├── All tools available              ├── Read-only tools only
├── Can read + write + execute       ├── Can only read + search + analyze
├── Changes applied immediately      ├── Produces a plan document
└── Full autonomy                    └── User reviews before execution
```

### Implementation

1. Maintain a mode flag in session state (`plan` vs `execute`)
2. In plan mode, filter the tool set to read-only tools
3. Add plan-specific tools: `CreatePlan`, `ExitPlanMode`
4. The plan output becomes input for execution mode

### When to Enter Plan Mode

- User explicitly asks to plan first
- Task is complex (touches many files/systems)
- Task is ambiguous (multiple valid approaches)
- Previous attempt failed (re-plan before retrying)

---

## Task Decomposition

Break complex tasks into discrete, actionable sub-tasks:

### Decomposition Strategy

1. **Understand the goal** — what does "done" look like?
2. **Identify dependencies** — which steps must come before others?
3. **Estimate scope** — how big is each step?
4. **Parallelize** — which steps can run concurrently?
5. **Order** — sequence the steps respecting dependencies

### Task Granularity

Tasks should be:
- **Atomic** — each task does one thing
- **Verifiable** — you can check if it's done
- **Independent** — minimal coupling between tasks (when possible)
- **Sized right** — one tool call to a few tool calls per task

---

## Todo and Task Tracking

### Simple Todo System (V1)

A flat list of todos with status tracking:

```
Todo {
  id: string
  content: string
  status: pending | in_progress | completed | cancelled
}
```

Agents create todos at the start of a complex task, mark them as they progress, and check for completeness before finishing.

### Hierarchical Task System (V2)

For more complex orchestration:

```
Task {
  id: string
  description: string
  status: pending | in_progress | completed | failed | cancelled
  subtasks: Task[]
  assignee?: AgentId      # For multi-agent delegation
  output?: string         # Result of completed task
  dependencies: TaskId[]  # Must complete before this task starts
}
```

### Task Lifecycle

```
pending → in_progress → completed
                      → failed → (retry or escalate)
                      → cancelled
```

---

## Mode Switching

Support transitions between different execution modes:

| From | To | Trigger |
|------|----|---------|
| Execute → Plan | Agent encounters complexity or ambiguity | Agent calls `EnterPlanMode` |
| Plan → Execute | Plan is ready and user approves | Agent calls `ExitPlanMode` |
| Execute → Review | Task is done, needs verification | Agent completes all todos |
| Any → Error | Unrecoverable failure | Error handler |

### Implementation

Mode switching is a tool the agent can call. The `EnterPlanMode` tool:
1. Saves current state
2. Restricts available tools to read-only
3. Adds plan-mode-specific tools
4. Signals the UI to show plan mode indicator

The `ExitPlanMode` tool:
1. Validates a plan was created
2. Restores full tool access
3. Optionally attaches the plan as context for execution

---

## Structured Reasoning

Help agents think through problems systematically:

### Chain-of-Thought Planning
Prompt the agent to explain its reasoning before each action. This surfaces mistakes early and creates an audit trail.

### Pre-Action Checklist
Before executing a multi-step plan, verify:
- [ ] All files to be modified have been read
- [ ] Dependencies between steps are respected
- [ ] Rollback strategy exists for destructive operations
- [ ] User has approved the overall approach

### Post-Action Verification
After each step, verify the outcome:
- Did the tool call succeed?
- Did the output match expectations?
- Should the plan be adjusted based on new information?

---

## Case Study: Claude Code

Claude Code implements several planning patterns:

**Plan Mode**: A dedicated mode where the agent can explore the codebase (read files, search, analyze) but cannot make changes. Entered via the `EnterPlanModeTool`, exited via `ExitPlanModeV2Tool` which produces a structured plan.

**Todo System**: The `TodoWriteTool` maintains a session-scoped todo list. Agents create todos at the start of complex tasks and update status as they work. The system enforces that agents don't end their turn with incomplete todos.

**Task Management (V2)**: A more sophisticated system with `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, and `TaskListTool`. Supports hierarchical tasks with assignees (for multi-agent delegation), dependencies, and status tracking.

**Mode Switching**: The agent can dynamically switch between plan and execute modes based on task complexity. The `hasExitedPlanMode` and `needsPlanModeExitAttachment` state flags ensure smooth transitions.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Plan enforcement | Always plan first | Agent decides | Agent decides — mandatory planning adds overhead for simple tasks |
| Todo persistence | Session only | Across sessions | Session — todos are contextual to the current task |
| Task granularity | One task per tool call | Logical grouping | Logical grouping — too granular creates noise |
| Plan format | Free-form text | Structured template | Structured for agent consumption, free-form for human review |

---

## Tags

#planning #reasoning #task-decomposition #todos #plan-mode #mode-switching #task-management #structured-reasoning #chain-of-thought #verification
