# Case Study: Claude Code

> An analysis of Claude Code's agent architecture, mapped to the universal patterns in this guide.

---

## Overview

Claude Code is a terminal-based AI coding assistant built as a React application running in a custom terminal UI framework. It implements virtually every pattern covered in the Agent Orchestration Guide at production scale:

| Metric | Value |
|--------|-------|
| TypeScript/TSX files | ~1,902 |
| Lines of code | ~800K+ |
| Slash commands | 100+ |
| Tools | 40+ |
| React hooks | 104 |
| Context window | 200K tokens |

---

## Pattern Mapping

How Claude Code implements each pattern from the guide:

### Agent Loop (Guide 01)

| Component | Implementation |
|-----------|----------------|
| Core loop | `query.ts` (69KB) — async query loop with streaming, tool execution, stop hooks |
| Query engine | `QueryEngine.ts` (46KB) — stateful wrapper for SDK/headless use |
| Entry points | `entrypoints/cli.tsx` → `main.tsx` → `replLauncher.tsx` |
| Streaming | Block-by-block parsing: text, thinking, tool_use events |
| Stop conditions | No tool calls, budget exceeded, max turns, user cancel, custom hooks |
| Cost tracking | `cost-tracker.ts` — per-model USD calculation from token counts |

### Tool System (Guide 02)

| Component | Implementation |
|-----------|----------------|
| Tool interface | `Tool.ts` — generic `Tool<Input, Output, Progress>` with ~20 fields |
| Factory | `buildTool()` fills defaults for simple tool definitions |
| Registry | `tools.ts` — `getAllBaseTools()`, `getTools()`, `assembleToolPool()` |
| Schema validation | Zod schemas for every tool input |
| Permission check | Per-tool `checkPermissions()` returning allow/ask/deny |
| Result mapping | `mapToolResultToToolResultBlockParam()` per tool |
| Presets | `TOOL_PRESETS`: minimal (3 tools), standard, full |

### Multi-Agent (Guide 03)

| Component | Implementation |
|-----------|----------------|
| Sub-agents | `AgentTool` → `spawnMultiAgent()` — isolated instances |
| Agent definitions | Markdown files with YAML frontmatter in `.claude/agents/` |
| Coordinator | `coordinator/coordinatorMode.ts` — plans and delegates |
| Teams | `TeamCreateTool`, `SendMessageTool` — shared mailbox communication |
| Task management | `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool` |
| Background agents | Output streams to terminal files, monitored via `TaskOutputTool` |
| Worktrees | `EnterWorktreeTool` / `ExitWorktreeTool` — git worktree isolation |

### Context & Memory (Guide 04)

| Component | Implementation |
|-----------|----------------|
| Token budget | `query/tokenBudget.ts` — tracks all context components |
| Micro-compact | `services/compact/microCompact.ts` — truncates oversized results |
| Auto-compact | `services/compact/autoCompact.ts` — summarizes at 80% threshold |
| Session memory | `services/SessionMemory/` — key facts persisted across compaction |
| Long-term memory | `memdir/` — markdown files in `~/.claude/memory/` |
| Memory consolidation | `services/autoDream/` — background consolidation during idle |
| Context assembly | System prompt + tools + history + injected context, ordered for caching |

### Permissions & Safety (Guide 05)

| Component | Implementation |
|-----------|----------------|
| Permission tiers | `isReadOnly()`, `isDestructive()` per tool |
| Permission context | `ToolPermissionContext` — mode, allow/deny/ask rules, working dirs |
| Permission modes | default, plan, auto, bypassPermissions, acceptEdits |
| Rule configuration | Layered settings: managed > local project > project > global |
| Read-before-write | `readFileState` map in `ToolUseContext` |
| Sandboxing | BashTool restricted environment, path validation |

### Human-in-the-Loop (Guide 06)

| Component | Implementation |
|-----------|----------------|
| Permission dialogs | `components/permissions/` — rich UI showing tool, input, risk |
| Ask-once | Session-scoped approval tracking |
| Trust dialog | `components/TrustDialog/` — first-run trust establishment |
| Structured questions | `AskUserQuestionTool` — multiple-choice user input |
| Plan mode | Read-only exploration before execution |

### Planning & Reasoning (Guide 07)

| Component | Implementation |
|-----------|----------------|
| Plan mode | `EnterPlanModeTool` / `ExitPlanModeV2Tool` — restrict to read-only |
| Todo system | `TodoWriteTool` — flat list with status tracking |
| Task management | V2 task CRUD with hierarchical tasks, assignees, dependencies |
| Mode switching | `hasExitedPlanMode`, `needsPlanModeExitAttachment` state flags |

### Protocols (Guide 09)

| Component | Implementation |
|-----------|----------------|
| MCP client | `MCPTool`, `McpAuthTool`, `ListMcpResourcesTool`, `ReadMcpResourceTool` |
| MCP server | `entrypoints/mcp.ts` — expose agent as MCP server |
| Bridge protocol | V1 (environment-based) + V2 (environment-less) |
| Transports | `WebSocketTransport`, `SSETransport`, `HybridTransport` |
| IDE integration | Direct-connect bridge for VS Code and JetBrains |
| JWT auth | `bridge/jwtUtils.ts` — session authentication |

### State & Lifecycle (Guide 10)

| Component | Implementation |
|-----------|----------------|
| Session state | `bootstrap/state.ts` — ~80 field singleton with accessor functions |
| Bootstrap | Multi-phase: args → config → migrate → trust → telemetry → tools → UI |
| Settings hierarchy | Managed > local project > project > global user > defaults |
| Migrations | Version-gated runner (`CURRENT_MIGRATION_VERSION = 11`) |
| Session lifecycle | UUID sessions, resume support, lineage via `parentSessionId` |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      USER INTERFACE                          │
│  Terminal (Ink TUI) ←→ React Components ←→ Hooks ←→ Context │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                     MAIN APPLICATION                         │
│  main.tsx → REPL → PromptInput → Commands (100+)            │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                     QUERY ENGINE                             │
│  query.ts → QueryEngine.ts → Tool Execution → Stop Hooks    │
│  Token Budget → Compaction → History → Cost Tracking         │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                     TOOL SYSTEM (40+)                        │
│  Files | Shell | Search | Agents | Tasks | Web | MCP | Plan │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                     SERVICES LAYER                           │
│  API Client | Analytics | Memory | Compact | Voice | MCP     │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                     TRANSPORT LAYER                          │
│  CLI (local) | Bridge (remote) | IDE Direct-Connect          │
│  WebSocket | SSE | Hybrid Transport                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **The agent loop is the foundation** — Claude Code's `query.ts` is the most important file. Everything else plugs into it.

2. **Tools are the capability layer** — 40+ tools with a consistent interface (`Tool.ts`), a registry (`tools.ts`), and the `buildTool()` factory for easy definition.

3. **Permissions enable trust** — Layered rules, per-tool permission checks, and human-in-the-loop approval make the system safe enough for real use.

4. **Context management is critical at scale** — Multi-stage compaction (micro → auto → session memory preservation) handles 200K token sessions.

5. **Multi-agent is a tool, not a framework** — Sub-agents are just another tool call (`AgentTool`). This makes the architecture simple and composable.

6. **State is a singleton** — One global state object (`bootstrap/state.ts`) with accessor functions. Simple, predictable, easy to reason about.

7. **Protocols enable extensibility** — MCP for dynamic tools, bridge for remote control, direct-connect for IDE integration. All built on the same message-passing patterns.

---

*This case study is based on analysis of the Claude Code source architecture. For detailed specs, see the original reference materials.*
