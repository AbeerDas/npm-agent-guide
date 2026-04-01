---
tags: [state, lifecycle, config, bootstrap, settings]
complexity: advanced
prerequisites: [00_foundations, 01_agent_loop]
---

# State & Lifecycle Management

> How to manage session state, bootstrap sequences, configuration hierarchies, and settings migrations in agent systems.

---

## Pattern (Universal)

Agent systems have complex state: session data, configuration, user preferences, model selection, cost tracking, and more. Managing this state cleanly — through startup, operation, and shutdown — determines whether your agent is reliable or brittle. This guide covers the state singleton pattern, layered configuration, bootstrap sequences, and version migrations.

---

## Session State Singleton

A single global state object that holds all per-session data. All components read from and write to this singleton.

### What Goes in Session State

| Category | Examples |
|----------|---------|
| Identity | Session ID, parent session ID, client type |
| Configuration | Current model, permission mode, feature flags |
| Metrics | Total cost, API duration, tokens used, lines changed |
| Working state | Current directory, read-file cache, active tools |
| Telemetry | Loggers, meters, counters, tracers |
| UI state | Active overlays, mode flags, pending operations |

### Design Principles

1. **Single source of truth** — no duplicated state across modules
2. **Accessor functions** — use getters/setters, not direct property access (enables validation and side effects)
3. **Leaf in import graph** — state module imports nothing from the main codebase (prevents circular dependencies)
4. **Reset capability** — support clearing state for testing and session reset

---

## Bootstrap Sequence

The startup sequence that initializes the agent from zero to ready:

### Typical Bootstrap Order

1. **Parse arguments** — CLI flags, environment variables
2. **Load configuration** — layered settings from files
3. **Initialize state** — create session state singleton
4. **Run migrations** — update settings from previous versions
5. **Security checks** — trust dialog, authentication
6. **Initialize telemetry** — analytics, logging, metrics
7. **Load tools** — discover and register built-in + MCP tools
8. **Load agents** — discover sub-agent definitions
9. **Setup UI** — render initial interface
10. **Prefetch** — async prefetch of data that will be needed (user context, tips, credentials)

### Fast-Path Optimization

Before loading the full application, check for fast-path commands that don't need the full bootstrap:
- `--version` → print and exit (zero imports)
- `--help` → print and exit
- Simple subcommands that don't need the full tool/agent system

---

## Configuration Hierarchy

Layer settings from multiple sources with clear precedence:

### Priority Order (highest to lowest)

| Layer | Source | Mutability | Scope |
|-------|--------|-----------|-------|
| 1. CLI flags | Command line arguments | Per-invocation | Session |
| 2. Managed/Enterprise | MDM/enterprise policy | Read-only | Machine |
| 3. Local project | `.agent/settings.local.json` | User-editable, gitignored | Project + user |
| 4. Project | `.agent/settings.json` | Shared, version-controlled | Project |
| 5. Global user | `~/.agent/settings.json` | User-editable | All projects |
| 6. Defaults | Hardcoded in application | Immutable | Universal |

### Resolution

For each setting, walk the layers from highest to lowest priority. First layer that defines the setting wins. This enables:
- Enterprise policies that users can't override
- Project-specific settings shared via version control
- Personal preferences that don't affect teammates
- Sensible defaults when nothing is configured

---

## Session Lifecycle

### Session Creation
1. Generate a unique session ID (UUID)
2. Initialize state singleton
3. Record start time, initial model, client type
4. Begin cost and metrics tracking

### Session Operation
- Track all API calls, tool executions, and state changes
- Support session resume (reload state from a previous session)
- Handle errors and recovery without losing state

### Session Termination
1. Flush pending metrics and telemetry
2. Persist session memory and history
3. Record final cost and duration
4. Clean up resources (MCP connections, background processes)

### Session Switching
Support switching between sessions (e.g., resuming a previous conversation):
1. Save current session state
2. Load target session state
3. Re-establish connections
4. Notify listeners of the switch

---

## Settings Migrations

As your agent evolves, settings formats change. Migrations ensure smooth upgrades:

### Migration Pattern

```
migrations = [
  { version: 1, up: renameOldSetting },
  { version: 2, up: moveSettingToNewLocation },
  { version: 3, up: convertSettingFormat },
]

function runMigrations(currentVersion, targetVersion):
  for migration in migrations[currentVersion..targetVersion]:
    migration.up()
  updateVersionNumber(targetVersion)
```

### Migration Best Practices

1. **Idempotent** — running a migration twice should be safe
2. **Non-destructive** — preserve old data when possible
3. **Versioned** — track the current migration version in config
4. **Logged** — record which migrations ran for debugging

---

## Case Study: Claude Code

Claude Code's state management includes:

**State Singleton**: `bootstrap/state.ts` holds ~80 fields in a single global state object. Accessor functions (`getSessionId()`, `getCwdState()`, `getTotalCostUSD()`) provide controlled access. The module is a strict leaf in the import DAG.

**Bootstrap**: Multi-phase startup: argument parsing → settings loading → migration → trust check → telemetry init → tool loading → agent loading → UI launch. Fast-paths for `--version`, `--dump-system-prompt`, and subcommands skip the full bootstrap.

**Configuration Hierarchy**: Four-layer settings (`managed > local project > project > global`) with pattern-based permission rules at each layer. Enterprise managed settings are read-only.

**Migrations**: Version-gated migration runner (`CURRENT_MIGRATION_VERSION = 11`) with named migrations for each settings format change (e.g., `migrateSonnet45ToSonnet46`, `migrateReplBridgeEnabledToRemoteControlAtStartup`).

**Session Lifecycle**: Sessions have unique IDs, support resume via session ID, track lineage via `parentSessionId`, and persist transcripts. The `regenerateSessionId()` function handles session reset with lineage tracking.

---

## Key Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| State management | Global singleton | Dependency injection | Singleton for simplicity, DI for testability — Claude Code uses singleton with test reset |
| Config format | JSON | YAML | JSON — universal support, no extra parser needed |
| Config storage | Single file | Multiple files | Multiple files per layer — enables gitignore for local settings |
| Migration strategy | In-place mutation | Copy-and-transform | In-place with backup — simpler, less storage |

---

## Tags

#state #lifecycle #config #bootstrap #settings #session-state #configuration-hierarchy #settings-migrations #session-lifecycle #bootstrap-sequence #fast-path #global-state
