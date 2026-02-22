# Cancel and Regenerate Examples

## Table of Contents

- [01 Cancel and Regenerate](#01-cancel-and-regenerate) - Stop running agents, regenerate failed runs, follow-up after cancel

## 01 Cancel and Regenerate

Source: `examples/cancel_regenerate/main.go`

Demonstrates starting a run with a slow tool, cancelling mid-execution, regenerating the cancelled run, and sending follow-up messages.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

// SlowTool simulates a long-running operation that respects context cancellation.
type SlowTool struct{}

func (t *SlowTool) Name() string        { return "slow_operation" }
func (t *SlowTool) Description() string { return "A slow operation that takes 30 seconds to complete." }

func (t *SlowTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"task": {Type: "string", Description: "Description of the task to perform"},
		},
		Required: []string{"task"},
	}
}

func (t *SlowTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct{ Task string `json:"task"` }
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}
	select {
	case <-time.After(30 * time.Second):
		return fmt.Sprintf("Completed: %s", params.Task), nil
	case <-ctx.Done():
		return "", ctx.Err()
	}
}

func main() {
	ctx := context.Background()
	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	client, _ := agentpg.NewClient(pgxv5.New(pool), &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"),
	})
	client.RegisterTool(&SlowTool{})
	client.Start(ctx)
	defer client.Stop(context.Background())

	agent, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "slow-assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Use slow_operation when asked.",
		Tools:        []string{"slow_operation"},
	})

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Step 1: Start run, wait for tool to begin
	runID, _ := client.RunFast(ctx, sessionID, agent.ID, "Do a slow 'data processing' operation", nil)
	time.Sleep(3 * time.Second)

	// Step 2: Cancel mid-execution
	client.CancelRun(ctx, runID)
	run, _ := client.GetRun(ctx, runID)
	fmt.Printf("State: %s\n", run.State) // "cancelled"

	// Step 3: Regenerate the cancelled run
	newRunID, _ := client.RegenerateRun(ctx, runID)
	response, _ := client.WaitForRun(ctx, newRunID)
	fmt.Println(response.Text)

	// Step 4: Continue the conversation
	followUp, _ := client.RunFastSync(ctx, sessionID, agent.ID, "What did you just do?", nil)
	fmt.Println(followUp.Text)
}
```
