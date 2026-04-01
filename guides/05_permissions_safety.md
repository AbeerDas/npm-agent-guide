---
tags: [permissions, safety, sandboxing, trust]
complexity: intermediate
prerequisites: [00_foundations, 02_tool_system]
---

# Permissions & Safety

> How to build permission models, sandboxing, and trust boundaries that let agents act powerfully while keeping humans in control.

---

## Pattern (Universal)

Agents that can execute tools need a permission system. Without one, every tool call is either fully trusted (dangerous) or requires manual approval (unusable). The solution is a tiered permission model with configurable rules that balance autonomy with safety.

---

## The Permission Tier Model

Four tiers of permission, from most to least autonomous:

| Tier | Behavior | Use For |
|------|----------|---------|
| **Automatic** | Execute without asking | Read-only operations, safe queries |
| **Ask-Once** | Ask user once, remember for session | File writes to known directories |
| **Ask-Always** | Ask user every time | Shell commands, destructive operations |
| **Deny** | Block completely | Operations outside trust boundary |

### How Tiers Map to Tools

Each tool declares its default permission tier. The system can override this based on:
- Input patterns (e.g., `rm -rf` always asks)
- File paths (e.g., writes to `/etc/` denied)
- User-configured rules
- Current permission mode

---

## Permission Modes

Support multiple modes for different contexts:

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Default** | Normal tier-based permissions | Interactive sessions |
| **Plan** | Read-only tools only, no writes | Planning/exploration phase |
| **Auto-accept** | Approve all edits automatically | Trusted automated workflows |
| **Bypass** | Skip all permission checks | Testing, CI/CD (use with caution) |

---

## Permission Rules

Rules are pattern-based matchers that override default tool permissions:

```
Rule = {
  tool: string | glob      # Which tool(s) this applies to
  path?: string | glob     # Optional path pattern
  action: allow | deny | ask
}
```

Rules are evaluated in priority order. First matching rule wins. If no rule matches, the tool's default permission applies.

### Rule Layering

Stack rules from multiple sources with clear precedence:

1. **Managed/Enterprise** (highest priority, read-only)
2. **Local project** (gitignored, per-developer)
3. **Project** (shared, checked into repo)
4. **Global user** (user-wide defaults)
5. **Tool defaults** (lowest priority)

---

## Rule-Based Access Control

### Allow Rules
```
allow: Bash(npm test)        # Allow specific commands
allow: FileWrite(src/**)     # Allow writes to src/
allow: WebFetch(api.example.com/**)  # Allow specific domains
```

### Deny Rules
```
deny: Bash(rm -rf *)         # Block destructive commands
deny: FileWrite(/etc/**)     # Block system file writes
deny: FileRead(.env)         # Block secret file reads
```

---

## Sandboxing Strategies

### Process-Level Sandboxing
Run tool executions in isolated processes with restricted syscalls. Limit filesystem access, network access, and resource usage.

### Directory Scoping
Restrict file operations to the project directory and explicitly allowed additional directories. Any path traversal outside these boundaries is denied.

### Network Allowlisting
For tools that make network requests, maintain an allowlist of permitted domains. Block all other outbound traffic by default.

### Resource Limits
Set timeouts, memory limits, and output size caps on tool executions to prevent runaway processes.

---

## Trust Boundaries

Define clear boundaries between trusted and untrusted contexts:

- **Trusted**: Agent's own reasoning, system prompt, user input
- **Semi-trusted**: Tool results, file contents, web content (may contain injection attempts)
- **Untrusted**: External API responses, MCP server data, plugin output

Never let untrusted content influence permission decisions directly.

---

## Destructive Operation Handling

Flag operations that can't be undone:

- File deletion
- Git force push
- Database drops
- System configuration changes

For destructive operations:
1. Always require explicit user confirmation
2. Show a preview of what will change
3. Provide an undo path when possible
4. Log the operation for audit

---

## Read-Before-Write Enforcement

Require that an agent reads a file before editing it. This prevents blind overwrites and ensures the agent has current context.

Implementation: Maintain a `readFileState` map tracking which files have been read (and their modification times). When a write/edit is requested, check that the file was recently read. If the file changed since the read, require a re-read.

---

## Case Study: Claude Code

Claude Code implements a comprehensive permission system:

**Permission Tiers**: Every tool declares `isReadOnly()` and `isDestructive()` flags. Read-only tools auto-approve; destructive tools always ask.

**Permission Context**: A `ToolPermissionContext` object carries the current mode, allow/deny/ask rules, and additional working directories through every tool call.

**Rule Configuration**: Rules stored in layered settings files (`~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`). Enterprise managed settings can enforce rules that users can't override.

**Sandboxing**: BashTool runs commands in a restricted environment. File tools validate paths against the project root and additional allowed directories.

**Permission Dialog**: When a tool needs approval, the UI shows a permission dialog with the tool name, input, and a description of what it will do. Users can approve once, approve always (add to allow rules), or deny.

---

## Code Template: TypeScript

See [templates/typescript/permission-system.ts](../templates/typescript/permission-system.ts) for a working implementation.

## Code Template: Python

See [templates/python/permission_system.py](../templates/python/permission_system.py) for a working implementation.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Default stance | Allow by default | Deny by default | Deny by default — safer, add allow rules as needed |
| Rule format | Glob patterns | Regex | Globs — simpler, sufficient for most cases |
| Rule storage | Config file | Database | Config file — auditable, version-controllable |
| Sandboxing depth | Process isolation | Container | Process isolation for dev, containers for production |

---

## Tags

#permissions #safety #sandboxing #trust #access-control #allow-rules #deny-rules #permission-tiers #permission-modes #read-before-write #destructive-operations #trust-boundaries
