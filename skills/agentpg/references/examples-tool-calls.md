# Tool Call Visibility Example

Source: `examples/tool_calls/main.go`

Demonstrates three ways to observe tool calls:
1. **Real-time callbacks** via `OnToolStart`/`OnToolComplete` on `RunOptions`
2. **Post-run inspection** via `response.ToolCalls`
3. **Query API** via `client.GetRunToolCalls(ctx, runID)`

## Full Example

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "os"
    "sync"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/youssefsiam38/agentpg"
    "github.com/youssefsiam38/agentpg/driver/pgxv5"
    "github.com/youssefsiam38/agentpg/tool"
)

// WeatherTool returns mock weather data.
type WeatherTool struct{}

func (t *WeatherTool) Name() string        { return "get_weather" }
func (t *WeatherTool) Description() string { return "Get the current weather for a city" }

func (t *WeatherTool) InputSchema() tool.ToolSchema {
    return tool.ToolSchema{
        Type: "object",
        Properties: map[string]tool.PropertyDef{
            "city": {Type: "string", Description: "City name"},
        },
        Required: []string{"city"},
    }
}

func (t *WeatherTool) Execute(_ context.Context, input json.RawMessage) (string, error) {
    var params struct {
        City string `json:"city"`
    }
    if err := json.Unmarshal(input, &params); err != nil {
        return "", fmt.Errorf("invalid input: %w", err)
    }
    time.Sleep(500 * time.Millisecond) // simulate latency
    return fmt.Sprintf("Weather in %s: 22°C, sunny with light clouds", params.City), nil
}

// TimeTool returns the current time.
type TimeTool struct{}

func (t *TimeTool) Name() string        { return "get_time" }
func (t *TimeTool) Description() string { return "Get the current date and time" }

func (t *TimeTool) InputSchema() tool.ToolSchema {
    return tool.ToolSchema{
        Type:       "object",
        Properties: map[string]tool.PropertyDef{},
    }
}

func (t *TimeTool) Execute(_ context.Context, _ json.RawMessage) (string, error) {
    return time.Now().Format(time.RFC1123), nil
}

func main() {
    ctx := context.Background()
    pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
    defer pool.Close()

    client, _ := agentpg.NewClient(pgxv5.New(pool), &agentpg.ClientConfig{
        APIKey: os.Getenv("ANTHROPIC_API_KEY"),
        Name:   "tool-calls-demo",
    })

    client.RegisterTool(&WeatherTool{})
    client.RegisterTool(&TimeTool{})
    client.Start(ctx)
    defer client.Stop(context.Background())

    agent, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
        Name:         "tool-calls-demo-agent",
        Model:        "claude-sonnet-4-5-20250929",
        SystemPrompt: "You are a helpful assistant. When asked about weather or time, always use the appropriate tool. Be concise.",
        Tools:        []string{"get_weather", "get_time"},
    })

    sessionID, _ := client.NewSession(ctx, nil, nil)

    // 1. Real-time callbacks
    var mu sync.Mutex
    var callbackEvents []string

    response, _ := client.RunFastSync(ctx, sessionID, agent.ID,
        "What's the weather in Cairo and what time is it?",
        &agentpg.RunOptions{
            OnToolStart: func(event agentpg.ToolCallEvent) {
                msg := fmt.Sprintf("Tool STARTED: %s", event.ToolName)
                fmt.Println(msg)
                mu.Lock()
                callbackEvents = append(callbackEvents, msg)
                mu.Unlock()
            },
            OnToolComplete: func(event agentpg.ToolCallEvent) {
                msg := fmt.Sprintf("Tool COMPLETED: %s (took %v, error=%v)",
                    event.ToolName, event.Duration.Round(time.Millisecond), event.IsError)
                fmt.Println(msg)
                mu.Lock()
                callbackEvents = append(callbackEvents, msg)
                mu.Unlock()
            },
        },
    )

    fmt.Printf("Response: %s\n", response.Text)

    // 2. Inspect ToolCalls on Response
    for i, tc := range response.ToolCalls {
        fmt.Printf("[%d] %s (iteration %d, state=%s, duration=%v)\n",
            i+1, tc.Name, tc.IterationNumber, tc.State, tc.Duration.Round(time.Millisecond))
        fmt.Printf("    Input:  %s\n", string(tc.Input))
        fmt.Printf("    Output: %s\n", tc.Output)
    }

    // 3. Query tool calls via GetRunToolCalls
    runID, _ := client.RunFast(ctx, sessionID, agent.ID, "What is the current time?", nil)
    queryResp, _ := client.WaitForRun(ctx, runID)

    toolCalls, _ := client.GetRunToolCalls(ctx, runID)
    fmt.Printf("Run %s: %d tool call(s)\n", runID, len(toolCalls))
    for _, tc := range toolCalls {
        fmt.Printf("  - %s → %s\n", tc.Name, tc.Output)
    }
    fmt.Printf("Response: %s\n", queryResp.Text)
}
```

## Key Points

- **`response.ToolCalls`** is populated automatically on every `Response` (best-effort; logged warning on failure, never fails the response)
- **Callbacks fire in goroutines** with panic recovery — they never block tool execution
- **Callbacks are in-memory only** — they only fire on the instance that executes the tool (not distributed)
- **Callbacks propagate to child runs** in agent-as-tool hierarchies — parent sees tool calls from the full hierarchy
- **`GetRunToolCalls(ctx, runID)`** and **`GetIterationToolCalls(ctx, iterationID)`** query the database directly
