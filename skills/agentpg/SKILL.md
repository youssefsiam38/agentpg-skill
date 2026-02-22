---
name: agentpg
description: "Build Go applications using the AgentPG framework—event-driven, PostgreSQL-backed AI agents with Claude integration. Use when working with AgentPG clients, agents, tools, sessions, runs, tool.Tool interface, agent hierarchies, distributed workers, compaction, admin UI, run variables, RunOptions, MCP server tools, cancel/regenerate runs, or transaction-safe APIs."
---

# AgentPG

Event-driven Go framework for async AI agents using PostgreSQL for state management. Source at `/home/youssef/projects/agentpg`.

## Workflow Decision Tree

1. **New project setup?** -> See "Quick Start" below
2. **Adding tools?** -> See [references/tools.md](references/tools.md)
3. **MCP server tools?** -> See [references/mcp.md](references/mcp.md)
4. **Agent hierarchies?** -> See [references/agents.md](references/agents.md)
5. **Cancel or regenerate runs?** -> See "Cancel and Regenerate" below
6. **Distributed workers / transactions / compaction / UI?** -> See [references/advanced.md](references/advanced.md)

## Quick Start

### Prerequisites

```bash
go get github.com/youssefsiam38/agentpg
go get github.com/youssefsiam38/agentpg/driver/pgxv5   # or driver/databasesql
# Optional: go get github.com/youssefsiam38/agentpg/mcp  # MCP server tools
psql $DATABASE_URL -f storage/migrations/001_agentpg_migration.up.sql
```

### Minimal Application

```go
package main

import (
    "context"
    "fmt"
    "log"
    "os"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/youssefsiam38/agentpg"
    "github.com/youssefsiam38/agentpg/driver/pgxv5"
)

func main() {
    ctx := context.Background()
    pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
    defer pool.Close()

    client, _ := agentpg.NewClient(pgxv5.New(pool), &agentpg.ClientConfig{
        APIKey: os.Getenv("ANTHROPIC_API_KEY"),
    })
    client.Start(ctx)
    defer client.Stop(context.Background())

    agent, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
        Name:         "assistant",
        Model:        "claude-sonnet-4-5-20250929",
        SystemPrompt: "You are a helpful assistant.",
    })

    sessionID, _ := client.NewSession(ctx, nil, nil)
    response, _ := client.RunFastSync(ctx, sessionID, agent.ID, "Hello!", nil)
    fmt.Println(response.Text)
}
```

### Lifecycle Order

1. `RegisterTool()` - before Start
2. `agentmcp.RegisterServer()` - MCP tools, before Start (optional)
3. `client.Start(ctx)` - starts background services
4. `CreateAgent()` / `GetOrCreateAgent()` - after Start (agents are DB entities)
5. `NewSession()` -> `Run*()` / `RunFast*()` - execute work
6. `client.Stop(ctx)` - graceful shutdown

## Core Patterns

### Run Execution

| Method | API | Behavior |
|--------|-----|----------|
| `Run()` | Batch | Async, returns run ID |
| `RunSync()` | Batch | Blocks until complete |
| `RunTx()` | Batch | With transaction |
| `RunFast()` | Streaming | Async, real-time |
| `RunFastSync()` | Streaming | Blocks, low latency |
| `RunFastTx()` | Streaming | With transaction |

All Run methods take: `(ctx, sessionID, agentID uuid.UUID, prompt string, opts *RunOptions)`.

Batch API = 50% cost discount, higher latency. Streaming = standard pricing, real-time.

### RunOptions

```go
type RunOptions struct {
    Variables            map[string]any  // Passed to tools via context
    OverrideInstructions string          // Replaces agent's system prompt
    AppendInstructions   string          // Appended to agent's system prompt
}
```

If `OverrideInstructions` is set, it replaces the system prompt entirely (and `AppendInstructions` is ignored). If only `AppendInstructions` is set, it's appended with `\n\n`. Pass `nil` for default behavior.

### Response Handling

```go
response, err := client.RunFastSync(ctx, sessionID, agent.ID, "Hello!", nil)
// response.Text         - final text
// response.StopReason   - "end_turn", "max_tokens", "tool_use"
// response.Usage        - InputTokens, OutputTokens
// response.Message      - full Message with Content []ContentBlock
```

### Run Variables and Instruction Overrides

Pass per-run data to tools and/or customize the system prompt:

```go
// Variables only
response, _ := client.RunSync(ctx, sessionID, agent.ID, "Process order", &agentpg.RunOptions{
    Variables: map[string]any{"order_id": "order-123", "tenant_id": "tenant-1"},
})

// Override system prompt for this run
response, _ := client.RunFastSync(ctx, sessionID, agent.ID, "Respond in French", &agentpg.RunOptions{
    OverrideInstructions: "You are a French-speaking assistant.",
})

// Append to system prompt
response, _ := client.RunFastSync(ctx, sessionID, agent.ID, "Hello!", &agentpg.RunOptions{
    AppendInstructions: "Always respond in bullet points.",
})
```

Variables are accessed in tools via context helpers - see [references/tools.md](references/tools.md).
Instruction overrides do NOT propagate to child runs; variables DO propagate.

### Cancel and Regenerate

```go
// Cancel a running agent (cascades to child runs, skips pending tools, cancels batch API)
err := client.CancelRun(ctx, runID)

// Regenerate a cancelled/failed run (deletes messages, creates fresh run with same params)
newRunID, err := client.RegenerateRun(ctx, runID)
response, _ := client.WaitForRun(ctx, newRunID)

// Continue conversation after cancel
newRunID, _ := client.RunFast(ctx, sessionID, agent.ID, "Do something else", nil)
```

`CancelRun` is idempotent (no-op on terminal runs), race-safe (`SELECT FOR UPDATE`), and unblocks `WaitForRun` callers. `RegenerateRun` only works on terminal runs.

### Model Selection

```go
"claude-opus-4-5-20251101"   // Most capable
"claude-sonnet-4-5-20250929" // Balanced (recommended default)
"claude-3-5-haiku-20241022"  // Fast and cheap
```

## Key Types

```go
// AgentDefinition - database entity with UUID
type AgentDefinition struct {
    ID           uuid.UUID
    Name         string            // Unique display name
    Description  string            // Shown when used as tool
    Model        string            // Claude model ID
    SystemPrompt string
    Tools        []string          // Tool names (must be registered on client)
    AgentIDs     []uuid.UUID       // Agent UUIDs for delegation
    MaxTokens    *int
    Temperature  *float64
    Metadata     map[string]any    // Multi-tenant isolation
}

// Response - from completed runs
type Response struct {
    Text           string
    StopReason     string
    Usage          Usage  // InputTokens, OutputTokens
    Message        *Message
    IterationCount int
    ToolIterations int
}

// Run states: pending -> batch_submitting -> batch_pending -> batch_processing
//             -> pending_tools -> completed/failed/cancelled
// Streaming:  pending -> streaming -> pending_tools -> completed/failed/cancelled
```

## ClientConfig Defaults

```go
MaxConcurrentRuns:  10
MaxConcurrentTools: 50
BatchPollInterval:  30s
RunPollInterval:    1s
ToolPollInterval:   500ms
HeartbeatInterval:  15s
LeaderTTL:          30s
StuckRunTimeout:    5min
```

## Error Handling

Sentinel errors: `ErrSessionNotFound`, `ErrRunNotFound`, `ErrAgentNotFound`, `ErrToolNotFound`, `ErrClientNotStarted`, `ErrInvalidStateTransition`, `ErrRunAlreadyFinalized`, `ErrRunNotCancellable`, `ErrToolExecutionFailed`, `ErrCompactionFailed`, `ErrStorageError`.

Tool errors: `tool.ToolCancel(err)` (no retry), `tool.ToolDiscard(err)` (permanent), `tool.ToolSnooze(duration, err)` (retry without consuming attempt), regular `error` (retried up to MaxAttempts).

## References

- **[references/tools.md](references/tools.md)** - Tool interface, schemas, FuncTool, context helpers, error types, database-aware tools
- **[references/mcp.md](references/mcp.md)** - MCP server tool integration (stdio/HTTP transports, namespacing, error mapping)
- **[references/agents.md](references/agents.md)** - Agent creation, delegation, multi-level hierarchies, agent-as-tool pattern
- **[references/advanced.md](references/advanced.md)** - Distributed workers, transactions, compaction, admin UI, monitoring, troubleshooting

## Examples

Full source code for all examples organized by topic. Consult the relevant file when building a specific feature:

- **[references/examples-basic.md](references/examples-basic.md)** - Getting started: simple chat, shared tools, database/sql driver, distributed workers
- **[references/examples-tools.md](references/examples-tools.md)** - Tool patterns: struct tools, FuncTool, schema validation, parallel execution
- **[references/examples-agents.md](references/examples-agents.md)** - Agent hierarchies: basic delegation, specialist agents, multi-level PM->Lead->Worker
- **[references/examples-compaction.md](references/examples-compaction.md)** - Context management: auto compaction, strategies, manual control, monitoring, extended context
- **[references/examples-ui.md](references/examples-ui.md)** - Admin UI: basic embedding, auth middleware, full-featured dashboard
- **[references/examples-retry.md](references/examples-retry.md)** - Error handling: instant retry, ToolCancel/Discard/Snooze error types, exponential backoff
- **[references/examples-cancel.md](references/examples-cancel.md)** - Cancel and regenerate: stop running agents, regenerate failed runs, continuation after cancel
