# Agent Orchestration Guide — Index

> Keyword → file/section lookup for agents and humans.
> Grep this file to find where any concept is documented.

---

## Guide Files

| # | File | Tags | What's Inside |
|---|------|------|---------------|
| 00 | [guides/00_foundations.md](guides/00_foundations.md) | `#foundations` `#mental-models` `#taxonomy` | What agents are, agent taxonomy, the universal agent loop, key abstractions |
| 01 | [guides/01_agent_loop.md](guides/01_agent_loop.md) | `#agent-loop` `#query-engine` `#execution-cycle` | The query-execute-observe cycle, streaming, stop conditions, turn management |
| 02 | [guides/02_tool_system.md](guides/02_tool_system.md) | `#tool-design` `#tool-registry` `#schemas` `#tool-execution` | Tool abstraction, input schemas, registry patterns, execution pipeline, result handling |
| 03 | [guides/03_multi_agent.md](guides/03_multi_agent.md) | `#multi-agent` `#orchestration` `#sub-agents` `#delegation` `#coordination` | Sub-agent spawning, orchestration topologies, coordinator mode, swarm, team systems |
| 04 | [guides/04_context_memory.md](guides/04_context_memory.md) | `#context-window` `#memory` `#compaction` `#token-budget` | Context window management, token budgets, compaction strategies, long-term memory, memory consolidation |
| 05 | [guides/05_permissions_safety.md](guides/05_permissions_safety.md) | `#permissions` `#safety` `#sandboxing` `#trust` | Permission tiers, rule-based access control, sandboxing, trust boundaries, deny rules |
| 06 | [guides/06_human_in_loop.md](guides/06_human_in_loop.md) | `#human-in-loop` `#approval` `#feedback` `#interactive` | Approval flows, permission dialogs, ask-once/ask-always, user feedback integration |
| 07 | [guides/07_planning_reasoning.md](guides/07_planning_reasoning.md) | `#planning` `#reasoning` `#task-decomposition` `#todos` | Plan mode, task management, todo systems, mode switching, structured reasoning |
| 08 | [guides/08_tool_catalog.md](guides/08_tool_catalog.md) | `#tool-catalog` `#file-tools` `#shell` `#search` `#web` | File read/write/edit, shell execution, glob/grep, web fetch/search, notebooks, code execution |
| 09 | [guides/09_protocols.md](guides/09_protocols.md) | `#protocols` `#mcp` `#bridge` `#transport` `#ide-integration` | Model Context Protocol, bridge protocols, WebSocket/SSE transports, IDE integration, remote sessions |
| 10 | [guides/10_state_lifecycle.md](guides/10_state_lifecycle.md) | `#state` `#lifecycle` `#config` `#bootstrap` `#settings` | Session state, bootstrap sequence, configuration layers, settings hierarchy, migrations |
| 11 | [guides/11_patterns_antipatterns.md](guides/11_patterns_antipatterns.md) | `#patterns` `#anti-patterns` `#best-practices` `#pitfalls` | Proven patterns, common mistakes, architecture trade-offs, scaling considerations |

---

## Keyword Lookup

### Agent Core

| Keyword | Guide | Section |
|---------|-------|---------|
| agent loop | 01 | §The Query-Execute-Observe Cycle |
| agent taxonomy | 00 | §Agent Taxonomy |
| agentic system | 00 | §What Is an Agent? |
| autonomy levels | 00 | §Autonomy Spectrum |
| bootstrap | 10 | §Bootstrap Sequence |
| core loop | 01 | §The Universal Agent Loop |
| entry point | 01 | §Entry Points and Initialization |
| execution cycle | 01 | §The Query-Execute-Observe Cycle |
| query engine | 01 | §Building a Query Engine |
| REPL | 01 | §Interactive vs Headless Modes |
| stop conditions | 01 | §Stop Conditions and Turn Boundaries |
| streaming | 01 | §Streaming Response Handling |
| turn management | 01 | §Turn Management |

### Tools

| Keyword | Guide | Section |
|---------|-------|---------|
| bash / shell tool | 08 | §Shell Execution Tools |
| buildTool | 02 | §The buildTool Pattern |
| code execution | 08 | §Code Execution Tools |
| concurrency safety | 02 | §Concurrency and Safety Flags |
| file edit | 08 | §File Tools |
| file read | 08 | §File Tools |
| file write | 08 | §File Tools |
| glob | 08 | §Search Tools |
| grep | 08 | §Search Tools |
| input schema | 02 | §Input Schemas and Validation |
| MCP tool | 09 | §Dynamic Tool Registration via MCP |
| notebook tool | 08 | §Notebook and Specialized Tools |
| search tools | 08 | §Search Tools |
| skill tool | 08 | §Extensible Tool Patterns |
| tool catalog | 08 | §(entire file) |
| tool definition | 02 | §The Tool Abstraction |
| tool execution | 02 | §Tool Execution Pipeline |
| tool framework | 02 | §(entire file) |
| tool permission | 02 | §Tool-Level Permissions |
| tool presets | 02 | §Tool Presets and Filtering |
| tool registry | 02 | §The Tool Registry |
| tool result | 02 | §Result Handling and Output Mapping |
| tool search | 02 | §Tool Discovery |
| tool validation | 02 | §Input Schemas and Validation |
| web fetch | 08 | §Web Tools |
| web search | 08 | §Web Tools |

### Multi-Agent

| Keyword | Guide | Section |
|---------|-------|---------|
| agent delegation | 03 | §Delegation Strategies |
| agent spawning | 03 | §Spawning Sub-Agents |
| background agents | 03 | §Background and Async Agents |
| coordinator mode | 03 | §The Coordinator Pattern |
| fan-out / fan-in | 03 | §Orchestration Topologies |
| multi-agent | 03 | §(entire file) |
| orchestration | 03 | §Orchestration Topologies |
| parallel agents | 03 | §Parallel Execution |
| sub-agent | 03 | §Spawning Sub-Agents |
| swarm mode | 03 | §Swarm Pattern |
| task management | 03 | §Task-Based Orchestration |
| team system | 03 | §Team-Based Collaboration |
| worker pool | 03 | §Swarm Pattern |

### Context & Memory

| Keyword | Guide | Section |
|---------|-------|---------|
| auto-compact | 04 | §Automatic Compaction |
| compaction | 04 | §Compaction Strategies |
| context injection | 04 | §Context Assembly |
| context window | 04 | §Context Window Fundamentals |
| conversation history | 04 | §Conversation History Management |
| long-term memory | 04 | §Long-Term Memory Systems |
| memory consolidation | 04 | §Memory Consolidation |
| memory directory | 04 | §File-Based Memory |
| micro-compact | 04 | §Micro-Compaction |
| session memory | 04 | §Session Memory |
| system prompt | 04 | §Context Assembly |
| token budget | 04 | §Token Budget Tracking |
| token estimation | 04 | §Token Counting and Estimation |

### Permissions & Safety

| Keyword | Guide | Section |
|---------|-------|---------|
| access control | 05 | §Rule-Based Access Control |
| allow / deny rules | 05 | §Permission Rules |
| auto-approve | 05 | §Automatic Approval |
| dangerous operations | 05 | §Destructive Operation Handling |
| deny rules | 05 | §Permission Rules |
| permission dialog | 06 | §Permission Dialogs |
| permission mode | 05 | §Permission Modes |
| permission tiers | 05 | §The Permission Tier Model |
| read-before-write | 05 | §Read-Before-Write Enforcement |
| sandboxing | 05 | §Sandboxing Strategies |
| trust boundary | 05 | §Trust Boundaries |
| trust dialog | 06 | §Initial Trust Establishment |

### Human-in-the-Loop

| Keyword | Guide | Section |
|---------|-------|---------|
| approval flow | 06 | §Approval Flow Architecture |
| ask-always | 06 | §Ask-Always vs Ask-Once |
| ask-once | 06 | §Ask-Always vs Ask-Once |
| feedback loop | 06 | §Feedback Integration |
| interactive mode | 06 | §Interactive Patterns |
| user confirmation | 06 | §Approval Flow Architecture |

### Planning & Reasoning

| Keyword | Guide | Section |
|---------|-------|---------|
| mode switching | 07 | §Mode Switching |
| plan mode | 07 | §Plan Mode Architecture |
| task creation | 07 | §Task Management Systems |
| task decomposition | 07 | §Task Decomposition |
| todo system | 07 | §Todo and Task Tracking |
| structured output | 07 | §Structured Reasoning |

### Protocols & Integration

| Keyword | Guide | Section |
|---------|-------|---------|
| bridge protocol | 09 | §Bridge Protocol Architecture |
| direct connect | 09 | §IDE Integration |
| HTTP transport | 09 | §Transport Layer Patterns |
| IDE integration | 09 | §IDE Integration |
| MCP (Model Context Protocol) | 09 | §Model Context Protocol |
| MCP resources | 09 | §MCP Resources |
| MCP server | 09 | §Running as an MCP Server |
| remote session | 09 | §Remote Sessions |
| SSE transport | 09 | §Transport Layer Patterns |
| WebSocket transport | 09 | §Transport Layer Patterns |

### State & Configuration

| Keyword | Guide | Section |
|---------|-------|---------|
| bootstrap state | 10 | §Bootstrap Sequence |
| config layers | 10 | §Configuration Hierarchy |
| global state | 10 | §Session State Singleton |
| migration | 10 | §Settings Migrations |
| session lifecycle | 10 | §Session Lifecycle |
| session state | 10 | §Session State Singleton |
| settings hierarchy | 10 | §Configuration Hierarchy |

### Patterns

| Keyword | Guide | Section |
|---------|-------|---------|
| anti-patterns | 11 | §Anti-Patterns |
| best practices | 11 | §(entire file) |
| error handling | 11 | §Error Handling Patterns |
| graceful degradation | 11 | §Resilience Patterns |
| idempotency | 11 | §Tool Idempotency |
| prompt engineering | 11 | §System Prompt Design |
| rate limiting | 11 | §Rate Limiting and Backoff |
| retry logic | 11 | §Resilience Patterns |
| scaling | 11 | §Scaling Considerations |

---

## Code Templates

| Template | Language | Path |
|----------|----------|------|
| Agent Loop | TypeScript | [templates/typescript/agent-loop.ts](templates/typescript/agent-loop.ts) |
| Agent Loop | Python | [templates/python/agent_loop.py](templates/python/agent_loop.py) |
| Tool Registry | TypeScript | [templates/typescript/tool-registry.ts](templates/typescript/tool-registry.ts) |
| Tool Registry | Python | [templates/python/tool_registry.py](templates/python/tool_registry.py) |
| Sub-Agent | TypeScript | [templates/typescript/sub-agent.ts](templates/typescript/sub-agent.ts) |
| Sub-Agent | Python | [templates/python/sub_agent.py](templates/python/sub_agent.py) |
| Context Manager | TypeScript | [templates/typescript/context-manager.ts](templates/typescript/context-manager.ts) |
| Context Manager | Python | [templates/python/context_manager.py](templates/python/context_manager.py) |
| Permission System | TypeScript | [templates/typescript/permission-system.ts](templates/typescript/permission-system.ts) |
| Permission System | Python | [templates/python/permission_system.py](templates/python/permission_system.py) |

---

## Case Studies

| System | Path | Key Patterns |
|--------|------|-------------|
| Claude Code | [case-studies/claude-code/](case-studies/claude-code/README.md) | Agent loop, 40+ tools, multi-agent, 200K context management, permission tiers, MCP integration |

---

*This index is kept in sync with guide content. When adding new guides, update this file.*
