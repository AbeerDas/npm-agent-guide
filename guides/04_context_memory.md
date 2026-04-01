---
tags: [context-window, memory, compaction, token-budget]
complexity: intermediate
prerequisites: [00_foundations, 01_agent_loop]
---

# Context & Memory Management

> How to manage finite context windows, implement compaction strategies, and build long-term memory systems for agents.

---

## Pattern (Universal)

Every LLM-based agent operates within a fixed context window. As conversations grow, agents must decide what to keep, what to summarize, and what to store externally. This guide covers the three layers of agent memory: working memory (context window), session memory (within a conversation), and long-term memory (across conversations).

---

## Context Window Fundamentals

The context window is your agent's working memory. It holds the system prompt, conversation history, tool definitions, and tool results. Managing it well is the difference between an agent that works on toy examples and one that handles real-world tasks.

Key constraints:
- **Fixed size**: Models have hard token limits (e.g., 128K, 200K tokens)
- **Cost scales linearly**: More tokens = more cost per API call
- **Prompt caching**: Keeping a stable prefix improves latency and cost
- **Quality degrades**: Models perform worse when context is cluttered with irrelevant information

---

## Token Budget Tracking

Track token usage across all components of your context:

| Component | Typical Budget Share | Priority |
|-----------|---------------------|----------|
| System prompt | 5-15% | Fixed, always present |
| Tool definitions | 10-20% | Semi-fixed, can filter |
| Conversation history | 40-60% | Dynamic, compactable |
| Tool results | 10-30% | Dynamic, truncatable |
| Reserved for response | 5-10% | Must be preserved |

### Implementation Pattern

Maintain a token budget tracker that monitors usage before each API call. When the total exceeds a threshold (e.g., 80% of max), trigger compaction.

---

## Context Assembly

Building the API request context follows a priority order:

1. **System prompt** — always included, defines agent behavior
2. **Tool definitions** — filtered to relevant tools when possible
3. **Recent history** — most recent turns are highest priority
4. **Older history** — summarized or compacted versions of earlier turns
5. **Injected context** — memory, project files, relevant documentation

---

## Compaction Strategies

### Full Compaction
Summarize the entire conversation history into a condensed form. Use when approaching context limits.

### Micro-Compaction
Selectively compress individual tool results or long messages without summarizing the whole conversation. More surgical, preserves more detail.

### Time-Based Compaction
Compress older messages more aggressively than recent ones. Recent context stays detailed; older context becomes summary.

### Grouped Compaction
Group related turns (e.g., a tool call and its result) and compress them together, preserving the logical structure.

---

## Conversation History Management

Maintain conversation history as a structured list of messages. Each message has a role (user, assistant, tool_result) and content. Key patterns:

- **Sliding window**: Keep the N most recent turns, discard older ones
- **Summary + recent**: Keep a running summary of older turns + full recent turns
- **Importance scoring**: Score each turn by relevance and keep the highest-scoring ones

---

## Session Memory

Within a single conversation, accumulate key facts, decisions, and context that the agent should remember:

- Files modified and their purposes
- User preferences discovered during the session
- Errors encountered and how they were resolved
- Architectural decisions made

Session memory can be injected into the system prompt or appended as context.

---

## Long-Term Memory Systems

### File-Based Memory
Store memories as markdown files in a well-known directory (e.g., `~/.agent/memory/`). Each memory is a file with metadata (timestamp, tags, relevance score). On session start, scan and inject relevant memories.

### Memory Consolidation
Periodically consolidate related memories into higher-level summaries. This prevents memory directories from growing unbounded while preserving key insights. Can be done as a background "dream" process during idle time.

### Relevance Scoring
When injecting memories into context, score each memory against the current task/conversation. Only inject memories above a relevance threshold.

---

## Case Study: Claude Code

Claude Code manages a 200K token context window across long coding sessions.

**Token Budget**: Tracks usage per-component (system prompt, tools, history, results) and triggers auto-compaction at ~80% capacity.

**Compaction Pipeline**: Uses a multi-stage approach:
1. `microCompact` — truncates oversized tool results in-place
2. `autoCompact` — triggers full conversation summarization when budget is exceeded
3. `sessionMemoryCompact` — preserves key session facts during compaction
4. `postCompactCleanup` — removes orphaned references after compaction

**Long-Term Memory (`memdir`)**: Stores memories as markdown files in `~/.claude/memory/`. Each file has YAML frontmatter with timestamps and tags. The `autoDream` service consolidates memories during idle periods, merging related memories and pruning stale ones.

**Session Memory**: Maintains a `SessionMemory` service that tracks important facts discovered during the session (e.g., project structure, test commands, coding conventions). These are persisted and re-injected on session resume.

---

## Code Template: TypeScript

See [templates/typescript/context-manager.ts](../templates/typescript/context-manager.ts) for a working implementation.

## Code Template: Python

See [templates/python/context_manager.py](../templates/python/context_manager.py) for a working implementation.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| When to compact | Fixed threshold (80%) | On-demand (when API rejects) | Fixed threshold — avoids failed requests |
| Compaction method | Full summarization | Sliding window | Summarization for long sessions, sliding window for short |
| Memory storage | Database | File system | Files for simplicity, DB for scale |
| Memory injection | Always inject all | Score and filter | Score and filter — keeps context relevant |

---

## Tags

#context-window #memory #compaction #token-budget #session-memory #long-term-memory #memory-consolidation #auto-compact #micro-compact #context-assembly
