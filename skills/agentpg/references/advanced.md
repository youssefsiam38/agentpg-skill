# AgentPG Advanced Features

## Table of Contents

- [Distributed Workers](#distributed-workers)
- [Transaction-Safe API](#transaction-safe-api)
- [Context Compaction](#context-compaction)
- [Admin UI](#admin-ui)
- [Run Rescue](#run-rescue)
- [LISTEN/NOTIFY Events](#listennotify-events)
- [Monitoring Queries](#monitoring-queries)
- [Troubleshooting](#troubleshooting)

## Distributed Workers

Multiple client instances share work from the same database using `SELECT FOR UPDATE SKIP LOCKED`.

```go
// Worker 1 (e.g., k8s pod 1)
client1, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
    APIKey: apiKey, Name: "worker", ID: "pod-1",
})
client1.RegisterTool(&LintTool{})
client1.RegisterTool(&TestTool{})
client1.Start(ctx)

// Worker 2 (e.g., k8s pod 2)
client2, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
    APIKey: apiKey, Name: "worker", ID: "pod-2",
})
client2.RegisterTool(&LintTool{})
client2.RegisterTool(&TestTool{})
client2.Start(ctx)
```

### Tool-Based Routing

Instances only claim work for tools they have registered:

```go
// Code worker - handles lint and test tools
codeWorker.RegisterTool(&LintTool{})
codeWorker.RegisterTool(&TestTool{})

// Data worker - handles query tools
dataWorker.RegisterTool(&QueryTool{})
dataWorker.RegisterTool(&ETLTool{})
```

Agents are database entities shared across all instances. Any instance with the required tools can process runs for any agent.

### Leader Election

One instance is automatically elected leader for maintenance tasks (cleanup stale instances, recover stuck runs).

```go
LeaderTTL:         30 * time.Second,  // Lease duration
HeartbeatInterval: 15 * time.Second,  // Refresh interval
```

## Transaction-Safe API

Atomically create application data and agent runs in a single transaction.

```go
func CreateOrderWithAgent(ctx context.Context, client *agentpg.Client, pool *pgxpool.Pool, order Order, agentID uuid.UUID) error {
    tx, _ := pool.Begin(ctx)
    defer tx.Rollback(ctx)

    // App data and agent run in same transaction
    orderID, _ := insertOrder(ctx, tx, order)
    sessionID, _ := client.NewSessionTx(ctx, tx, nil, map[string]any{"order_id": orderID})
    runID, _ := client.RunTx(ctx, tx, sessionID, agentID,
        fmt.Sprintf("Process order %s", orderID),
        &agentpg.RunOptions{Variables: map[string]any{"order_id": orderID}},
    )

    tx.Commit(ctx)              // Commit first
    client.WaitForRun(ctx, runID) // Then wait
    return nil
}
```

**No `RunSyncTx`**: Would deadlock because the run is not visible until the transaction commits, but RunSyncTx would wait before committing.

Transaction methods: `NewSessionTx()`, `RunTx()`, `RunFastTx()`.

## Context Compaction

Manage long conversations by compacting context when approaching model limits.

### Auto Compaction

```go
client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
    AutoCompactionEnabled: true,
    CompactionConfig: &compaction.Config{
        Strategy:          compaction.StrategyHybrid,  // or StrategySummarization
        Trigger:           0.85,       // Compact at 85% context usage
        TargetTokens:      80000,
        PreserveLastN:     10,         // Keep last 10 messages
        ProtectedTokens:   40000,      // Protect recent tokens
        MaxTokensForModel: 200000,
        SummarizerModel:   "claude-3-5-haiku-20241022",
        SummarizerMaxTokens: 4096,
    },
})
```

### Manual Compaction

```go
needsCompaction, _ := client.NeedsCompaction(ctx, sessionID)
stats, _ := client.GetCompactionStats(ctx, sessionID)
// stats.CurrentTokens, stats.TriggerThreshold, stats.NeedsCompaction

result, _ := client.Compact(ctx, sessionID)
result, _ := client.CompactIfNeeded(ctx, sessionID)  // Only if threshold reached
result, _ := client.CompactWithConfig(ctx, sessionID, customConfig)
// result.OriginalTokens, result.CompactedTokens, result.MessagesRemoved
```

### Strategies

- **Hybrid** (default): Phase 1 prunes tool outputs, Phase 2 summarizes if still over target
- **Summarization**: Directly summarizes all compactable messages using Claude

### Protected Messages

Messages marked `is_preserved=true` or `is_summary=true` are never compacted. Last N messages and messages within `ProtectedTokens` are also preserved.

## Admin UI

```go
import "github.com/youssefsiam38/agentpg/ui"

uiConfig := &ui.Config{
    BasePath:            "/ui",
    PageSize:            25,
    RefreshInterval:     5 * time.Second,
    MetadataFilter:      map[string]any{"tenant_id": "my-tenant"},
    MetadataFilterKeys:  []string{"tenant_id", "user_id"},
    MetadataDisplayKeys: []string{"tenant_id", "user_id"},
    ReadOnly:            false,
}

http.Handle("/ui/", http.StripPrefix("/ui", ui.UIHandler(drv.Store(), client, uiConfig)))
```

### Pages

| Path | Description |
|------|-------------|
| `/dashboard` | Overview stats, recent runs, active instances |
| `/sessions`, `/sessions/{id}` | Session list and detail |
| `/runs`, `/runs/{id}` | Run list with iterations and tool executions |
| `/runs/{id}/conversation` | Full conversation view |
| `/tool-executions` | Tool execution queue |
| `/agents` | Database agents with capable instances |
| `/instances` | Worker instance health |
| `/compaction` | Compaction event history |
| `/chat`, `/chat/session/{id}` | Interactive chat interface |

## Run Rescue

Automatically recover stuck runs.

```go
client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
    StuckRunTimeout: 5 * time.Minute,
    RunRescueConfig: &agentpg.RunRescueConfig{
        RescueInterval:    time.Minute,
        RescueTimeout:     5 * time.Minute,
        MaxRescueAttempts: 3,
    },
})
```

The leader instance periodically checks for stuck runs and resets them to `pending` for re-processing.

## LISTEN/NOTIFY Events

| Channel | Trigger |
|---------|---------|
| `agentpg_run_created` | New pending run |
| `agentpg_run_state` | Run state change |
| `agentpg_run_finalized` | Completed/failed/cancelled |
| `agentpg_tool_pending` | New tool execution ready |
| `agentpg_tools_complete` | All tools for a run done |

Polling fallbacks: `RunPollInterval` (1s), `ToolPollInterval` (500ms), `BatchPollInterval` (30s).

## Monitoring Queries

```sql
-- Active runs by state
SELECT state, COUNT(*) FROM agentpg_runs WHERE finalized_at IS NULL GROUP BY state;

-- Tool execution queue
SELECT tool_name, COUNT(*) FROM agentpg_tool_executions WHERE state = 'pending' GROUP BY tool_name;

-- Instance health
SELECT id, name, active_run_count, last_heartbeat_at FROM agentpg_instances ORDER BY last_heartbeat_at DESC;

-- Batch API status
SELECT batch_id, batch_status, batch_poll_count FROM agentpg_iterations WHERE batch_status = 'in_progress';
```

## Troubleshooting

### Run Stuck in Pending

```sql
-- Check agent exists and get its required tools
SELECT id, name, tools FROM agentpg_agents WHERE id = 'agent-uuid';

-- Check which instances have the required tools
SELECT it.instance_id, it.tool_name FROM agentpg_instance_tools it
WHERE it.tool_name IN (SELECT unnest(tools) FROM agentpg_agents WHERE id = 'agent-uuid');

-- Check instance health
SELECT id, last_heartbeat_at FROM agentpg_instances ORDER BY last_heartbeat_at DESC;
```

### Tool Execution Not Starting

Verify the tool is registered on at least one running instance:
```sql
SELECT it.instance_id, it.tool_name FROM agentpg_instance_tools it WHERE it.tool_name = 'tool-name';
SELECT id, tool_name, state, created_at FROM agentpg_tool_executions WHERE state = 'pending' ORDER BY created_at;
```

### Common Issues

- **"ErrClientNotStarted"**: Call `client.Start(ctx)` before creating agents or running
- **"ErrAgentNotFound"**: Use agent UUID (not name) in Run calls. Create agent after Start
- **"ErrToolNotFound"**: Register tool on client before Start, and include in agent's `Tools` array
- **Deadlock with RunSyncTx**: Use `RunTx()` + `WaitForRun()` instead (commit tx first, then wait)
