---
tags: [patterns, anti-patterns, best-practices, pitfalls]
complexity: advanced
prerequisites: [00_foundations]
---

# Patterns & Anti-Patterns

> Proven patterns that make agents reliable and common mistakes that make them fail. Distilled from production agent systems.

---

## Pattern (Universal)

Building agent systems involves recurring design decisions. This guide catalogs patterns that consistently work well and anti-patterns that consistently cause problems. Use it as a checklist when designing or reviewing agent architectures.

---

## Proven Patterns

### 1. Read-Before-Write

Always read the current state before modifying it. For files: read the file before editing. For configs: read the current value before updating. This prevents blind overwrites and ensures the agent has accurate context.

**Implementation**: Track reads in a map (`path → { content, mtime }`). Before any write, check that the target was recently read and hasn't changed since.

### 2. Graceful Degradation

When a tool call fails, the agent should adapt — not crash. Return structured error messages that the agent can reason about and retry or work around.

**Pattern**: Tool results always include success/failure status. On failure, include the error message, possible causes, and suggested fixes.

### 3. Idempotent Operations

Design tools so that running them twice with the same input produces the same result. This makes retries safe and simplifies error recovery.

**Example**: FileEdit with exact string matching — if the old string was already replaced, the operation fails safely ("string not found") rather than making duplicate changes.

### 4. Progressive Disclosure of Capability

Don't expose all 40+ tools to the agent at once. Start with a core set and let the agent discover more via `ToolSearchTool` or context-based filtering.

**Implementation**: Tool presets (`minimal`, `standard`, `full`) that can be selected based on task complexity. A `ToolSearchTool` that searches the full catalog by description.

### 5. Prompt-Cache-Friendly Ordering

When building the API request, keep the stable parts (system prompt, tool definitions) at the beginning. Dynamic content (conversation history, tool results) goes at the end. This maximizes prompt cache hit rates and reduces cost.

### 6. Structured Tool Results

Tool results should be structured data, not free-form strings. Include metadata (success/failure, affected files, line numbers) that the agent can parse and reason about.

### 7. Conversation Checkpointing

Periodically checkpoint the conversation state so you can recover from failures without losing progress. This is especially important for long-running agent sessions.

### 8. Layered Validation

Validate at multiple levels:
1. **Schema validation** — input types and required fields (via Zod/JSON Schema)
2. **Semantic validation** — does the input make sense? (e.g., file path exists)
3. **Permission validation** — is this operation allowed?
4. **Runtime validation** — did the operation succeed?

---

## Anti-Patterns

### 1. Unbounded Context Growth

**Problem**: Appending every tool result to context without limits. Context fills up, costs increase, and model performance degrades.

**Fix**: Implement token budgets, result size limits, and automatic compaction. Truncate large tool results (e.g., cap at 10K characters).

### 2. Permission Fatigue

**Problem**: Asking the user for approval on every operation. Users start auto-approving without reading, defeating the purpose of permissions.

**Fix**: Use tiered permissions. Auto-approve read-only operations. Remember approval decisions (ask-once). Group similar approvals.

### 3. Blind Writes

**Problem**: Writing to files the agent hasn't read. Results in overwrites, lost content, and incorrect assumptions.

**Fix**: Enforce read-before-write. Track which files have been read and their modification times. Reject writes to unread files.

### 4. Infinite Tool Loops

**Problem**: Agent keeps calling tools in a loop without making progress. Common when error handling causes retries of the same failing operation.

**Fix**: Implement stop conditions: max turns per query, max consecutive tool calls, budget limits. Detect repeated tool calls with identical inputs.

### 5. Monolithic System Prompt

**Problem**: Stuffing everything into one massive system prompt. Hard to maintain, wastes tokens on irrelevant instructions.

**Fix**: Modular system prompt assembly. Include base instructions always, add context-specific sections based on the current task/mode.

### 6. Serialized Tool Execution

**Problem**: Running all tools sequentially when many could run in parallel. Wastes time, especially for read-only operations.

**Fix**: Check tool concurrency safety. Run read-only, concurrency-safe tools in parallel. Serialize only tools that modify state.

### 7. Stateless Sub-Agents

**Problem**: Sub-agents that lose all context when they complete. Important findings, decisions, and context vanish.

**Fix**: Sub-agents should return structured summaries. Parent agents should capture and preserve key findings from sub-agent work.

### 8. Hardcoded Tool Sets

**Problem**: Agent can only use tools that were compiled in. No extensibility, no dynamic capabilities.

**Fix**: Support dynamic tool registration (via MCP or plugin system). Let tools be discovered at runtime.

---

## Error Handling Patterns

### Retry with Backoff
For transient failures (network errors, rate limits), implement exponential backoff with jitter:
- Start: 1 second
- Max: 60 seconds
- Jitter: +/- 50%
- Max retries: 3-5

### Error Classification
Classify errors to determine the right response:
- **Transient** → retry (network timeout, rate limit)
- **Input error** → fix input and retry (invalid path, wrong format)
- **Permission error** → escalate to user (insufficient permissions)
- **Fatal** → abort and report (unrecoverable state)

### Graceful Error Messages
When returning errors to the agent, include:
1. What happened (error type and message)
2. Why it might have happened (common causes)
3. What to try next (suggested fixes)

---

## System Prompt Design

### Do
- Be specific about the agent's role and capabilities
- Include examples of correct behavior
- Define what the agent should NOT do
- Set tone and style guidelines
- Include context about the environment (OS, project type, etc.)

### Don't
- Include instructions for every possible scenario (too long)
- Use vague language ("be helpful" — every LLM already tries this)
- Contradict yourself across sections
- Include dynamic content in the stable prompt prefix (breaks caching)

---

## Resilience Patterns

### Circuit Breaker
If a service fails repeatedly, stop calling it for a cooldown period. Prevents cascading failures.

### Timeout Everything
Every external call should have a timeout. Every tool execution should have a timeout. No operation should be allowed to hang indefinitely.

### Resource Limits
Set explicit limits on:
- Memory usage per tool execution
- Output size per tool result
- Number of concurrent operations
- Total cost per session

---

## Rate Limiting and Backoff

### API Rate Limiting
Track usage against provider rate limits. When approaching limits:
1. Show user a warning with wait time
2. Queue requests and drip-feed them
3. Fall back to a lower-cost model if available

### Self-Imposed Rate Limiting
Prevent your own agent from overwhelming external services:
- Limit concurrent web fetches
- Throttle file system operations in rapid succession
- Cap MCP calls per second

---

## Scaling Considerations

### Horizontal Scaling (Multiple Agent Sessions)
- Each session should be self-contained (no shared mutable state between sessions)
- Use unique session IDs for all logging and metrics
- Support concurrent sessions without interference

### Vertical Scaling (More Capable Sessions)
- Token budgets scale with model capability (larger models, larger context)
- Tool pools can grow with demand (dynamic registration)
- Background agents for parallelism within a session

### Cost Scaling
- Track per-session cost in real-time
- Support cost budgets and alerts
- Use model fallbacks (expensive model fails → try cheaper model)
- Cache and reuse results when possible

---

## Tool Idempotency

Design tools so that identical inputs produce identical results when called multiple times:

| Tool | Idempotent? | Notes |
|------|-------------|-------|
| FileRead | Yes | Same file → same content (if not modified) |
| FileWrite | Yes | Writing same content twice → same result |
| FileEdit | Conditional | First call succeeds, second fails ("not found") — safely idempotent |
| Bash | No | Side effects vary (rm, mkdir, etc.) |
| WebFetch | Mostly | Content may change between calls |
| Search | Yes | Same query → same results (if index unchanged) |

---

## Case Study: Claude Code

Claude Code embodies many of these patterns:

**Read-Before-Write**: The `readFileState` map enforces that files are read before editing. FileEditTool checks the map and rejects edits to unread files.

**Graceful Degradation**: All tool calls return structured `ToolResult` objects with typed data. Errors include descriptions and are fed back to the model for reasoning.

**Prompt Caching**: Tool definitions and system prompts are ordered for stability. The `assembleToolPool()` function sorts tools by name for consistent prompt prefix.

**Progressive Disclosure**: `ToolSearchTool` lets the agent discover tools by description when the full set isn't loaded. `TOOL_PRESETS` provide `minimal`, `standard`, and `full` sets.

**Error Handling**: The `withRetry()` utility implements exponential backoff with jitter. Error classification drives the retry strategy. Rate limit errors show countdown timers.

**Resilience**: Every API call has a timeout. Tool executions have per-tool timeouts. The token budget system prevents context overflow.

---

## Tags

#patterns #anti-patterns #best-practices #pitfalls #error-handling #retry #rate-limiting #idempotency #system-prompt #resilience #scaling #read-before-write #graceful-degradation #progressive-disclosure #prompt-caching
