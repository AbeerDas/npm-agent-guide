---
tags: [multi-agent, orchestration, sub-agents, delegation, coordination]
complexity: intermediate
prerequisites: [00_foundations, 01_agent_loop, 02_tool_system]
---

# Multi-Agent Orchestration

> How to spawn sub-agents, implement orchestration topologies, build coordinator patterns, and manage teams of agents working together.

---

## Why Multi-Agent?

A single agent has limits:
- **Context window** — one agent can only hold so much context
- **Specialization** — different tasks benefit from different prompts/tools/models
- **Parallelism** — sequential execution is slow for independent tasks
- **Isolation** — risky experiments shouldn't pollute the main agent's state

Multi-agent systems solve these by delegating work to specialized sub-agents that operate with their own context, tools, and constraints.

---

## Spawning Sub-Agents

A sub-agent is an independent agent instance spawned by a parent agent to handle a specific task.

### The AgentTool Pattern

Implement sub-agent spawning as a tool that the parent agent can call:

```
AgentTool {
  name: "Agent"
  input: {
    prompt: string              // Task description for the sub-agent
    agentType?: string          // Which agent definition to use
    allowedTools?: string[]     // Tool subset for the sub-agent
    model?: string              // Model override
    permissionMode?: string     // Permission restrictions
  }
  
  execution:
    1. Create a new agent loop instance
    2. Configure with specified tools, model, permissions
    3. Inject the prompt as user input
    4. Run the agent loop to completion
    5. Return the sub-agent's final response to the parent
}
```

### Sub-Agent Isolation

Sub-agents should be isolated from the parent:
- **Own context window** — starts fresh, doesn't inherit parent's full history
- **Own tool set** — can be restricted (e.g., read-only sub-agent)
- **Own permission mode** — can be more or less restrictive than parent
- **Shared filesystem** — operates on the same codebase
- **Cost tracking** — costs roll up to the parent session

### What to Pass to Sub-Agents

| Pass | Don't Pass |
|------|-----------|
| Task description | Full conversation history |
| Relevant context snippets | Unrelated context |
| File paths to work with | All session memory |
| Constraints and requirements | Parent's internal state |

The key insight: give sub-agents **focused context**. They perform better with a clear task and minimal noise than with the parent's full history.

---

## Orchestration Topologies

### 1. Hub-and-Spoke (Coordinator)

One central coordinator delegates to specialist agents:

```
                    ┌──────────────┐
                    │  Coordinator  │
                    └──────┬───────┘
               ┌───────────┼───────────┐
               ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  Agent A  │ │  Agent B  │ │  Agent C  │
        │ (backend) │ │(frontend) │ │ (tests)   │
        └──────────┘ └──────────┘ └──────────┘
```

- Coordinator plans the work and delegates
- Each agent handles a specific domain
- Coordinator aggregates results
- Best for: heterogeneous tasks with clear domain boundaries

### 2. Pipeline (Sequential)

Agents process in sequence, each building on the previous:

```
Agent A → Agent B → Agent C → Result
(plan)    (implement)  (review)
```

- Each agent receives the output of the previous
- Best for: tasks with natural phases (plan → implement → test → review)
- Risk: errors compound through the pipeline

### 3. Fan-Out / Fan-In (Parallel)

Parent spawns multiple agents working in parallel on independent tasks:

```
              ┌──────────┐
              │  Parent   │
              └─────┬─────┘
         ┌──────────┼──────────┐
         ▼          ▼          ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │  Agent 1  │ │  Agent 2  │ │  Agent 3  │
   │ (file A)  │ │ (file B)  │ │ (file C)  │
   └──────┬───┘ └──────┬───┘ └──────┬───┘
         └──────────┼──────────┘
                    ▼
              ┌──────────┐
              │  Parent   │
              │ (merge)   │
              └──────────┘
```

- All agents work simultaneously
- Parent waits for all to complete, then merges results
- Best for: independent tasks (editing different files, running different tests)
- Risk: merge conflicts if agents touch overlapping areas

### 4. Swarm (Worker Pool)

Many identical worker agents process a queue of similar tasks:

```
                    ┌──────────────┐
                    │  Controller   │
                    │  (task queue) │
                    └──────┬───────┘
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │ Worker 1  │     │ Worker 2  │     │ Worker 3  │
   │ (task X)  │     │ (task Y)  │     │ (task Z)  │
   └──────────┘     └──────────┘     └──────────┘
```

- Controller distributes homogeneous tasks to a pool of workers
- Workers are identical (same tools, model, prompt)
- Best for: batch processing (e.g., apply the same refactor across many files)

---

## The Coordinator Pattern

A coordinator is a specialized agent whose primary job is to plan and delegate, not to execute directly.

### Coordinator Design

```
CoordinatorAgent {
  systemPrompt: "You are a coordinator. Your job is to:
    1. Understand the user's request
    2. Break it into sub-tasks
    3. Delegate each sub-task to an appropriate agent
    4. Monitor progress
    5. Aggregate results
    You should NOT execute tasks directly — delegate to specialists."
  
  tools: [
    AgentTool,        // Spawn sub-agents
    TaskCreateTool,   // Track sub-tasks
    TaskGetTool,      // Check task status
    FileReadTool,     // Read for planning context
    GrepTool,         // Search for planning context
  ]
  
  // Restricted — no write tools
}
```

### When to Use a Coordinator

- Task spans multiple files or systems
- Task benefits from parallel execution
- Different parts need different specializations
- The overall context would exceed a single agent's window

### When NOT to Use a Coordinator

- Task is simple and fits in one agent's context
- The overhead of coordination exceeds the benefit
- Tasks are tightly coupled (agents would conflict)

---

## Delegation Strategies

### By Domain

Delegate based on what part of the system the task touches:

| Sub-Task Domain | Delegate To |
|----------------|-------------|
| Backend API changes | Backend specialist agent |
| Frontend UI changes | Frontend specialist agent |
| Database migrations | Database specialist agent |
| Test writing | Test specialist agent |

### By Capability

Delegate based on what skills are needed:

| Capability Needed | Agent Configuration |
|-------------------|-------------------|
| Code generation | Full tool access, code-focused prompt |
| Code review | Read-only tools, review-focused prompt |
| Research | Web search + file read tools only |
| Testing | Shell + file read tools, test-focused prompt |

### By Risk Level

Delegate risky operations to agents with appropriate restrictions:

| Risk Level | Agent Configuration |
|------------|-------------------|
| Low (read-only) | Auto-approve all, no restrictions |
| Medium (writes) | Ask-once permission mode |
| High (shell, destructive) | Ask-always, restricted tools |

---

## Background and Async Agents

Some tasks are long-running and shouldn't block the main agent:

### Background Agent Pattern

```
function spawnBackgroundAgent(task):
  agent = createAgent(task)
  outputFile = createTemporaryFile()
  
  // Run in background — output streams to file
  runInBackground(agent, outputFile)
  
  return {
    agentId: agent.id,
    outputFile: outputFile.path,
    status: "running",
  }
```

### Monitoring Background Agents

The parent agent can check on background agents:
- **TaskOutputTool** — read the current output of a background agent
- **TaskStopTool** — cancel a background agent
- **Polling** — periodically check if the agent has completed

### Output Streaming

Background agents write their output to a file that can be tailed:
- Header: agent ID, PID, start time
- Body: agent output as it streams
- Footer: exit code, elapsed time (when complete)

---

## Task-Based Orchestration

For complex multi-agent work, use a task management system:

### Task Lifecycle

```
Task {
  id: string
  description: string
  status: pending | in_progress | completed | failed | cancelled
  assignee?: AgentId
  result?: string
  dependencies: TaskId[]
}
```

### Orchestration Flow

1. Coordinator creates tasks with dependencies
2. Tasks with no dependencies are assigned to agents
3. Agents work on their tasks and update status
4. When a task completes, check if dependent tasks can start
5. When all tasks complete, coordinator aggregates results

---

## Team-Based Collaboration

For agents that need to communicate directly with each other (not just through a coordinator):

### Team Model

```
Team {
  id: string
  name: string
  members: AgentId[]
  mailbox: Message[]       // Shared message queue
}
```

### Inter-Agent Communication

```
SendMessageTool {
  input: { teamId: string, message: string }
  // Delivers message to team mailbox
  // Other agents on the team can read it
}
```

### When to Use Teams

- Agents working on tightly coupled tasks
- Need to share discoveries without going through coordinator
- Real-time coordination (e.g., one agent writes code, another writes tests simultaneously)

---

## Parallel Execution

### Identifying Parallelizable Work

Tasks are parallelizable when they:
- Touch different files
- Have no data dependencies
- Don't conflict in their effects

### Conflict Prevention

When running agents in parallel on the same codebase:
1. **Git worktrees** — each agent gets its own branch/worktree
2. **File locking** — track which files each agent is modifying
3. **Merge strategy** — reconcile changes after parallel work completes
4. **Conflict detection** — check for overlapping modifications before merging

### Worktree Pattern

```
function parallelWithWorktrees(tasks):
  for task in tasks:
    branch = createBranch(task.name)
    worktree = git.worktree.add(branch)
    agent = createAgent(task, cwd=worktree.path)
    agents.append(agent)
  
  // Run all agents in parallel
  results = await Promise.all(agents.map(a => a.run()))
  
  // Merge results back
  for result in results:
    git.merge(result.branch)
```

---

## Case Study: Claude Code

Claude Code implements a comprehensive multi-agent system:

**AgentTool**: Spawns sub-agents via `spawnMultiAgent()`. Each sub-agent gets its own context window, tool set, and permission context. The parent receives the sub-agent's final response as a tool result. Sub-agents can be configured with custom agent definitions (stored as markdown with YAML frontmatter in `.claude/agents/`).

**Agent Definitions**: Agents are defined as markdown files:
```yaml
---
agentType: "backend-specialist"
description: "Handles backend API changes"
tools: ["FileRead", "FileEdit", "Bash", "Grep"]
model: "claude-sonnet-4-6"
permissionMode: "default"
memory: "session"
---
System prompt content here...
```

**Coordinator Mode**: A dedicated mode (`coordinator/coordinatorMode.ts`) where the main agent orchestrates work across multiple sub-agents. The coordinator plans, delegates, and aggregates.

**Swarm Mode**: Parallel worker agents for batch processing. Multiple agents work in parallel, each in their own context.

**Team System**: `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool` — create teams of agents that can communicate via a shared mailbox. The `context/mailbox.tsx` provides the mailbox implementation.

**Task Management**: `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool` — full task CRUD with status tracking, assignees, and dependencies. `TaskStopTool` and `TaskOutputTool` for managing background agents.

**Background Agents**: Long-running agents write output to terminal files. The parent can monitor progress, read output, or cancel via `TaskOutputTool` and `TaskStopTool`.

**Worktree Support**: `EnterWorktreeTool` and `ExitWorktreeTool` enable agents to work in isolated git worktrees. Combined with tmux, enables parallel agents in separate worktrees.

---

## Code Template: TypeScript

See [templates/typescript/sub-agent.ts](../templates/typescript/sub-agent.ts) for a working implementation.

## Code Template: Python

See [templates/python/sub_agent.py](../templates/python/sub_agent.py) for a working implementation.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Sub-agent context | Inherit parent history | Start fresh | Start fresh — cleaner context, better focus |
| Agent isolation | Process-level | Thread-level | Process — true isolation, but higher overhead |
| Coordination | Centralized coordinator | Peer-to-peer teams | Coordinator for most cases, teams for tightly coupled work |
| Parallel merge | Auto-merge | Human review | Auto-merge with conflict detection, human review for conflicts |
| Agent definitions | Hardcoded | File-based (YAML/MD) | File-based — extensible, user-configurable |
| Cost tracking | Per-agent | Roll up to parent | Both — per-agent for debugging, rolled up for billing |

---

## Tags

#multi-agent #orchestration #sub-agents #delegation #coordination #coordinator #swarm #fan-out #fan-in #pipeline #team #background-agents #parallel-execution #worktree #task-management #agent-definitions #agent-tool
