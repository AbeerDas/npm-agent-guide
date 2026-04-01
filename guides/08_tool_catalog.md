---
tags: [tool-catalog, file-tools, shell, search, web]
complexity: intermediate
prerequisites: [02_tool_system]
---

# Tool Catalog

> Reference catalog of common tool patterns for agent systems: file operations, shell execution, search, web access, and specialized tools.

---

## Pattern (Universal)

Most agent systems need a common set of capabilities: reading and writing files, executing commands, searching codebases, and accessing the web. This catalog documents proven implementations for each tool category, covering input schemas, permission requirements, and edge cases.

---

## File Tools

### FileReadTool

Read file contents with optional line range selection.

| Property | Value |
|----------|-------|
| Permission | Read-only (auto-approve) |
| Concurrency | Safe |
| Key inputs | `path`, `offset?`, `limit?` |

Design considerations:
- Support partial reads (line offset + limit) for large files
- Return line numbers in output for precise referencing
- Handle binary files gracefully (detect and skip or convert)
- Normalize paths and resolve symlinks
- Track reads in a `readFileState` map for read-before-write enforcement

### FileWriteTool

Create new files or overwrite existing ones.

| Property | Value |
|----------|-------|
| Permission | Ask (creates/overwrites files) |
| Concurrency | Not safe |
| Key inputs | `path`, `contents` |

Design considerations:
- Require read-before-write for existing files
- Create parent directories automatically
- Validate path is within allowed directories
- Show diff preview in approval dialog

### FileEditTool

Make targeted edits to existing files using find-and-replace.

| Property | Value |
|----------|-------|
| Permission | Ask (modifies files) |
| Concurrency | Not safe |
| Key inputs | `path`, `old_string`, `new_string` |

Design considerations:
- Require `old_string` to uniquely match in the file (fail if ambiguous)
- Preserve indentation and line endings
- Support a `replace_all` flag for bulk replacements
- More surgical than FileWrite — doesn't require rewriting the whole file

---

## Shell Execution Tools

### BashTool

Execute shell commands and capture output.

| Property | Value |
|----------|-------|
| Permission | Ask-always (commands can be destructive) |
| Concurrency | Depends on command |
| Key inputs | `command`, `working_directory?`, `timeout?` |

Design considerations:
- Set timeouts to prevent hanging commands
- Capture both stdout and stderr
- Support background execution for long-running processes (dev servers, watchers)
- Maintain shell state (cwd, env vars) across sequential calls
- Sandbox: restrict filesystem access, network access, resource usage
- Parse commands for known-dangerous patterns (rm -rf, force push, etc.)

---

## Search Tools

### GlobTool

Find files by name pattern.

| Property | Value |
|----------|-------|
| Permission | Read-only (auto-approve) |
| Concurrency | Safe |
| Key inputs | `pattern`, `directory?` |

### GrepTool

Search file contents with regex.

| Property | Value |
|----------|-------|
| Permission | Read-only (auto-approve) |
| Concurrency | Safe |
| Key inputs | `pattern`, `path?`, `include?`, `context_lines?` |

Design considerations for both:
- Respect `.gitignore` patterns by default
- Limit result count to prevent overwhelming context
- Return enough surrounding context for results to be useful
- Support file type filtering

---

## Web Tools

### WebFetchTool

Fetch a URL and return its contents as readable text.

| Property | Value |
|----------|-------|
| Permission | Ask or allowlisted domains |
| Concurrency | Safe |
| Key inputs | `url` |

Design considerations:
- Convert HTML to markdown for readability
- Strip navigation, ads, scripts — extract main content
- Set reasonable size limits on response
- Handle redirects, timeouts, and error pages gracefully
- Domain allowlisting for automatic approval

### WebSearchTool

Search the web for information.

| Property | Value |
|----------|-------|
| Permission | Ask or auto-approve |
| Concurrency | Safe |
| Key inputs | `query` |

---

## Notebook and Specialized Tools

### NotebookEditTool
Edit Jupyter notebook cells. Handle the JSON structure internally, present cell contents to the agent.

### LSPTool
Interface with Language Server Protocol for type checking, go-to-definition, diagnostics.

### REPLTool
Execute code in an interactive REPL session (Python, Node, etc.) with state persistence across calls.

---

## Extensible Tool Patterns

### SkillTool
User-defined macro commands that combine multiple tool calls into a single reusable action.

### MCPTool
Dynamically registered tools from external MCP servers. The agent discovers available tools at runtime.

### ToolSearchTool
Meta-tool that searches the available tool catalog for relevant capabilities. Useful when the agent has many tools and needs to find the right one.

---

## Case Study: Claude Code

Claude Code implements 40+ tools across all categories:

**File Tools**: `FileReadTool`, `FileWriteTool`, `FileEditTool` — with read-before-write enforcement via `readFileState` tracking. FileEdit uses exact string matching for surgical edits.

**Shell**: `BashTool` with timeout management, background process support (outputs stream to terminal files), and stateful cwd/env persistence. Commands are analyzed for destructive patterns.

**Search**: `GlobTool` and `GrepTool` built on ripgrep for performance. Results capped to prevent context overflow. Support recursive searching with gitignore awareness.

**Web**: `WebFetchTool` converts pages to markdown, `WebSearchTool` returns summarized results. Both have domain-based permission rules.

**Specialized**: `NotebookEditTool` for Jupyter cells, `SkillTool` for user-defined macros, `MCPTool` for dynamic MCP tools, `ToolSearchTool` for tool discovery when the full set is too large to include in context.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| File edit approach | Full file rewrite | Find-and-replace | Find-and-replace — less error-prone for targeted changes |
| Shell state | Stateless per call | Stateful across calls | Stateful — matches human expectations |
| Search result limits | Hard cap | Dynamic based on context budget | Dynamic — adapts to available space |
| Web content processing | Raw HTML | Markdown conversion | Markdown — much more useful for LLMs |

---

## Tags

#tool-catalog #file-tools #shell #search #web #file-read #file-write #file-edit #bash #glob #grep #web-fetch #web-search #notebook #mcp-tool #skill-tool
