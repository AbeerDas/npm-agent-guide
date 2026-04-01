---
tags: [agent-loop, query-engine, execution-cycle]
complexity: foundational
prerequisites: [00_foundations]
---

# The Agent Loop

> The query-execute-observe cycle that powers every agent system. How to build it, manage turns, handle streaming, and implement stop conditions.

---

## The Query-Execute-Observe Cycle

The agent loop is the core execution cycle of any agent system. It has three phases that repeat until a stop condition is met:

1. **Query** — Assemble context and call the LLM
2. **Execute** — Parse the response and run any tool calls
3. **Observe** — Feed tool results back into context and decide whether to continue

```
         ┌──────────────────────────────────┐
         │           USER INPUT              │
         └──────────────┬───────────────────┘
                        ▼
         ┌──────────────────────────────────┐
         │        ASSEMBLE CONTEXT           │
         │  System prompt + history + tools  │
         └──────────────┬───────────────────┘
                        ▼
         ┌──────────────────────────────────┐
    ┌───▶│          CALL LLM API            │
    │    │  Stream response tokens           │
    │    └──────────────┬───────────────────┘
    │                   ▼
    │    ┌──────────────────────────────────┐
    │    │        PARSE RESPONSE             │
    │    │                                   │
    │    │  ├── Text block → accumulate      │
    │    │  ├── Thinking block → log/display │
    │    │  └── Tool use block → execute     │
    │    └──────────────┬───────────────────┘
    │                   ▼
    │    ┌──────────────────────────────────┐
    │    │    TOOL EXECUTION (if needed)     │
    │    │                                   │
    │    │  1. Validate input                │
    │    │  2. Check permissions              │
    │    │  3. Execute tool                   │
    │    │  4. Format result                  │
    │    └──────────────┬───────────────────┘
    │                   ▼
    │    ┌──────────────────────────────────┐
    │    │       CHECK STOP CONDITIONS       │
    │    │                                   │
    │    │  ├── No tool calls → DONE         │
    │    │  ├── Budget exceeded → DONE       │
    │    │  ├── Max turns → DONE             │
    │    │  └── More tool calls → CONTINUE   │
    │    └──────────────┬───────────────────┘
    │                   │
    │         ┌─────────┴─────────┐
    │         │ CONTINUE          │ DONE
    └─────────┘                   ▼
                       ┌──────────────────┐
                       │  RETURN RESPONSE  │
                       └──────────────────┘
```

---

## Entry Points and Initialization

Before the loop runs, the system needs to bootstrap:

### Interactive Mode (REPL)
1. Parse CLI arguments and load configuration
2. Initialize the UI (terminal, IDE panel, web interface)
3. Load tools, agents, and MCP connections
4. Display the prompt and wait for user input
5. On input: enter the agent loop

### Headless Mode (Print/Script)
1. Parse arguments (prompt comes from `-p` flag or stdin)
2. Load tools (potentially a minimal set)
3. Run the agent loop once
4. Print the result and exit

### SDK/Programmatic Mode
1. Initialize the query engine with configuration
2. Call `query(prompt, options)` directly
3. Receive structured results

The key insight: **the agent loop itself is the same** regardless of how it's invoked. The entry point just determines how input arrives and output is presented.

---

## Building a Query Engine

The query engine is the stateful component that manages the agent loop. It holds configuration, conversation history, and the tool pool.

### Core Components

```
QueryEngine {
  // Configuration
  model: string                    // Which LLM to use
  systemPrompt: string             // Base instructions
  tools: Tool[]                    // Available tools
  maxTokens: number                // Context window size
  
  // State
  history: Message[]               // Conversation history
  tokenBudget: TokenBudget         // Usage tracking
  costTracker: CostTracker         // Financial tracking
  
  // Methods
  query(input: string): Response   // Run one agent loop
  addMessage(msg: Message): void   // Append to history
  compact(): void                  // Compress history
}
```

### The Query Method (Pseudocode)

```
function query(userInput):
  // 1. Add user message to history
  history.append({ role: "user", content: userInput })
  
  // 2. Assemble the API request
  request = {
    model: this.model,
    system: this.systemPrompt,
    messages: this.history,
    tools: this.tools.map(t => t.toAPISchema()),
    max_tokens: calculateResponseBudget(),
  }
  
  // 3. Call the LLM
  response = await llmClient.stream(request)
  
  // 4. Process the response
  assistantMessage = { role: "assistant", content: [] }
  toolResults = []
  
  for block in response:
    if block.type == "text":
      assistantMessage.content.append(block)
      
    if block.type == "tool_use":
      assistantMessage.content.append(block)
      result = await executeTool(block.name, block.input)
      toolResults.append({
        type: "tool_result",
        tool_use_id: block.id,
        content: result,
      })
  
  // 5. Add assistant message to history
  history.append(assistantMessage)
  
  // 6. If there were tool calls, feed results back and continue
  if toolResults.length > 0:
    history.append({ role: "user", content: toolResults })
    
    // Check stop conditions before recursing
    if not shouldStop():
      return query()  // Continue the loop (no new user input)
  
  // 7. Track cost
  costTracker.add(response.usage)
  
  // 8. Check if compaction is needed
  if tokenBudget.shouldCompact():
    compact()
  
  return response
```

---

## Streaming Response Handling

Production agents stream responses from the LLM rather than waiting for the full response. This provides real-time feedback and enables progressive rendering.

### Stream Processing Pipeline

```
LLM API → Stream Events → Parser → Renderer + Tool Executor
```

### Event Types

| Event | Action |
|-------|--------|
| `message_start` | Initialize response object, capture metadata |
| `content_block_start` | Begin accumulating a text, thinking, or tool_use block |
| `content_block_delta` | Append text deltas, update UI in real-time |
| `content_block_stop` | Finalize the block; if tool_use, queue for execution |
| `message_delta` | Capture stop reason, usage stats |
| `message_stop` | Response complete, process all queued tool calls |

### Streaming + Tool Execution

When the LLM streams a tool use block, you have two options:

1. **Wait for message completion** — collect all tool calls, then execute them (possibly in parallel). Simpler to implement.
2. **Execute as they arrive** — start executing tool calls as each block completes, while the LLM continues streaming. Lower latency, more complex.

Most systems use option 1 for simplicity. Option 2 is an optimization for latency-sensitive applications.

---

## Turn Management

A "turn" is one cycle of the agent loop: user input → LLM response → tool execution → result. Turns are how you track progress and enforce limits.

### Turn Counting

| What Counts as a Turn | What Doesn't |
|----------------------|--------------|
| Each LLM API call | UI rendering |
| Each tool execution | History reads |
| Each permission check | Internal calculations |

### Multi-Turn Conversations

In a multi-turn conversation, the agent loop runs once per user message. But each loop iteration may itself involve multiple "inner turns" (LLM calls tools, results fed back, LLM calls more tools, etc.).

```
User Turn 1: "Read the package.json and tell me the dependencies"
  └── Inner Turn 1: LLM calls FileRead("package.json")
  └── Inner Turn 2: LLM receives file content, responds with text

User Turn 2: "Update the lodash version to 5.0"
  └── Inner Turn 1: LLM calls FileRead("package.json")  [read before write]
  └── Inner Turn 2: LLM calls FileEdit("package.json", old, new)
  └── Inner Turn 3: LLM receives edit confirmation, responds with summary
```

---

## Stop Conditions and Turn Boundaries

The agent loop must have clear stop conditions to prevent runaway execution:

### Stop Condition Hierarchy

| Priority | Condition | Action |
|----------|-----------|--------|
| 1 (highest) | User cancels (Ctrl+C, abort signal) | Immediate stop |
| 2 | Cost budget exceeded | Stop after current tool completes |
| 3 | Token budget exceeded | Trigger compaction or stop |
| 4 | Max turns reached | Stop and return partial result |
| 5 | No more tool calls | Natural completion |
| 6 | Stop hook fires | Custom stop logic (e.g., test passed) |

### Implementing Stop Hooks

Stop hooks are custom functions that run after each turn to decide if the agent should stop. Useful for goal-directed agents:

```
stopHooks = [
  // Stop when all tests pass
  (turnResult) => turnResult.toolName == "Bash" 
    && turnResult.output.includes("All tests passed"),
    
  // Stop when a specific file exists
  (turnResult) => fs.existsSync("output/result.json"),
]
```

---

## Context Assembly

Before each LLM call, assemble the full context:

### Assembly Order (for prompt caching)

1. **System prompt** (stable — cacheable)
2. **Tool definitions** (semi-stable — cacheable when sorted)
3. **Injected context** (project files, memory — changes per session)
4. **Conversation history** (changes every turn)
5. **Pending tool results** (latest turn only)

Keep the stable parts at the beginning of the request. This maximizes prompt cache hit rates and reduces cost and latency.

### Token Budget Allocation

```
Total budget: 200,000 tokens (example)

System prompt:   ~5,000 tokens   (2.5%)
Tool definitions: ~20,000 tokens (10%)
Reserved for response: ~10,000   (5%)
Available for history: ~165,000  (82.5%)
```

When history exceeds its allocation, trigger compaction (see [04_context_memory.md](04_context_memory.md)).

---

## Interactive vs Headless Modes

### Interactive (REPL)
- User types input, agent responds, repeat
- Full UI with streaming output, permission dialogs, status indicators
- Session persists until user exits
- Supports commands (e.g., `/compact`, `/plan`, `/help`)

### Headless (Print/Script)
- Single prompt in, single result out
- No UI, no dialogs — all permissions must be pre-configured
- Used in CI/CD, automation scripts, batch processing
- May have `max_turns` limit for safety
- Output format configurable (text, JSON, streaming JSON)

### SDK (Programmatic)
- Library integration: your code calls the query engine directly
- Full control over lifecycle, tools, and event handling
- No terminal UI — your application provides the interface
- Typically used for building custom agent products

---

## Error Handling in the Loop

### API Errors
- **Rate limit** → exponential backoff with jitter, show countdown
- **Overloaded** → retry with longer delay
- **Auth failure** → re-authenticate, retry once
- **Invalid request** → log, report to user, don't retry

### Tool Execution Errors
- **Permission denied** → return structured error, let LLM decide next step
- **Timeout** → kill the process, return timeout error
- **Invalid input** → return validation error with details
- **Runtime error** → capture stack trace, return as tool result

### Recovery Strategy
Always feed errors back to the LLM as tool results. The LLM can often self-correct:
- Wrong file path → LLM searches for the right path
- Command fails → LLM reads the error and tries a different approach
- Permission denied → LLM tries a different tool or asks the user

---

## Cost Tracking

Track cost per session to enable budgeting and billing:

### What to Track

| Metric | Source |
|--------|--------|
| Input tokens | API response `usage.input_tokens` |
| Output tokens | API response `usage.output_tokens` |
| Cache read tokens | API response `usage.cache_read_input_tokens` |
| Cache creation tokens | API response `usage.cache_creation_input_tokens` |
| Cost in USD | Calculated from token counts × model pricing |
| API call count | Counter per model |
| API duration | Wall clock time per call |
| Tool duration | Wall clock time per tool execution |

### Budget Enforcement

Set a `max_budget_usd` and check after each API call. When exceeded, stop the loop and return the partial result with a budget warning.

---

## Case Study: Claude Code

Claude Code's agent loop is implemented across two files:

**`query.ts` (69KB)** — The main async query loop function. Handles:
- Context assembly (system prompt + history + tools + token budget)
- Streaming API calls via the Claude client
- Tool execution dispatch (validates, checks permissions, executes, maps results)
- Stop hook orchestration (runs custom stop conditions after each turn)
- Cost tracking (input/output/cache tokens, USD calculation)
- Auto-compaction (triggers `microCompact` and `autoCompact` when budget is tight)

**`QueryEngine.ts` (46KB)** — A stateful class wrapping the query loop for SDK/headless use. Provides:
- `submitQuery(prompt)` — run one agent loop
- Dependency injection for tools, history, permissions
- Event emission for progress tracking
- Abort controller integration for cancellation

**Key implementation details:**
- The loop handles up to 200K tokens of context
- Tool results are size-capped (`maxResultSizeChars`) and truncated if too large
- Stop hooks run after each inner turn, checked via `query/stopHooks.ts`
- Token budget tracked by `query/tokenBudget.ts` which monitors all context components
- Streaming events are parsed block-by-block; tool_use blocks are queued and executed after the message completes
- Supports both interactive mode (Ink TUI renders streaming output) and headless mode (`-p` flag, outputs text/JSON)
- Cost tracked by `cost-tracker.ts` which computes USD from per-model pricing tables

**Stop conditions in Claude Code:**
1. No more tool calls in the response (natural completion)
2. `stop_reason: "end_turn"` from the API
3. Token budget exceeded → auto-compact then retry
4. Max turns reached (configurable via `--max-turns`)
5. Cost budget exceeded (`--max-budget-usd`)
6. User cancellation (SIGINT handler)
7. Custom stop hooks (e.g., stop when tests pass)

---

## Code Template: TypeScript

See [templates/typescript/agent-loop.ts](../templates/typescript/agent-loop.ts) for a working implementation.

## Code Template: Python

See [templates/python/agent_loop.py](../templates/python/agent_loop.py) for a working implementation.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Loop control | Recursive function calls | While loop with state | While loop — avoids stack overflow on long sessions |
| Tool execution timing | After message completes | As tool blocks stream in | After message — simpler, group parallel-safe tools |
| Context assembly | Rebuild every turn | Incremental append | Rebuild — ensures consistency, enables compaction |
| Error handling | Throw and catch | Return error as tool result | Return as tool result — LLM can self-correct |
| Streaming | Always stream | Batch for headless | Always stream — consistent code path, better UX |

---

## Tags

#agent-loop #query-engine #execution-cycle #streaming #stop-conditions #turn-management #context-assembly #cost-tracking #error-handling #interactive #headless #sdk
