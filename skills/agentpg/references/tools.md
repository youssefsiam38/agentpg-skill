# AgentPG Tools Reference

## Table of Contents

- [Tool Interface](#tool-interface)
- [ToolSchema and PropertyDef](#toolschema-and-propertydef)
- [FuncTool Convenience](#functool-convenience)
- [Run Context Helpers](#run-context-helpers)
- [Tool Error Types](#tool-error-types)
- [Database-Aware Tool](#database-aware-tool)
- [Registration and Agent Binding](#registration-and-agent-binding)

## Tool Interface

```go
import "github.com/youssefsiam38/agentpg/tool"

type Tool interface {
    Name() string
    Description() string
    InputSchema() ToolSchema
    Execute(ctx context.Context, input json.RawMessage) (string, error)
}
```

### Struct-Based Tool

```go
type WeatherTool struct {
    apiKey string
}

func (t *WeatherTool) Name() string        { return "get_weather" }
func (t *WeatherTool) Description() string { return "Get current weather for a city" }

func (t *WeatherTool) InputSchema() tool.ToolSchema {
    return tool.ToolSchema{
        Type: "object",
        Properties: map[string]tool.PropertyDef{
            "city": {Type: "string", Description: "City name"},
            "unit": {Type: "string", Description: "celsius or fahrenheit", Enum: []string{"celsius", "fahrenheit"}},
        },
        Required: []string{"city"},
    }
}

func (t *WeatherTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
    var params struct {
        City string `json:"city"`
        Unit string `json:"unit"`
    }
    if err := json.Unmarshal(input, &params); err != nil {
        return "", fmt.Errorf("invalid input: %w", err)
    }
    // ... implementation
    return fmt.Sprintf("Weather in %s: 22C, sunny", params.City), nil
}
```

## ToolSchema and PropertyDef

```go
type ToolSchema struct {
    Type        string                       // Must be "object"
    Properties  map[string]PropertyDef
    Required    []string
    Description string
}

type PropertyDef struct {
    Type              string                 // "string", "number", "integer", "boolean", "array", "object"
    Description       string
    Enum              []string               // Allowed values
    Default           any
    Minimum           *float64
    Maximum           *float64
    MinLength         *int
    MaxLength         *int
    Pattern           string                 // Regex pattern
    Items             *PropertyDef           // For arrays
    MinItems          *int
    MaxItems          *int
    Properties        map[string]PropertyDef // For nested objects
    Required          []string               // For nested objects
}
```

### Schema Examples

**Array property:**
```go
"tags": {Type: "array", Description: "List of tags", Items: &tool.PropertyDef{Type: "string"}, MaxItems: intPtr(10)}
```

**Nested object:**
```go
"address": {
    Type: "object",
    Properties: map[string]tool.PropertyDef{
        "street": {Type: "string"},
        "city":   {Type: "string"},
    },
    Required: []string{"street", "city"},
}
```

**Number with constraints:**
```go
"count": {Type: "integer", Minimum: floatPtr(1), Maximum: floatPtr(100)}
```

## FuncTool Convenience

For simple tools without state:

```go
myTool := tool.NewFuncTool(
    "greet",
    "Greet a user by name",
    tool.ToolSchema{
        Type: "object",
        Properties: map[string]tool.PropertyDef{
            "name": {Type: "string", Description: "User's name"},
        },
        Required: []string{"name"},
    },
    func(ctx context.Context, input json.RawMessage) (string, error) {
        var p struct{ Name string `json:"name"` }
        json.Unmarshal(input, &p)
        return fmt.Sprintf("Hello, %s!", p.Name), nil
    },
)
client.RegisterTool(myTool)
```

## Run Context Helpers

Access per-run variables and metadata inside tool Execute:

```go
import "github.com/youssefsiam38/agentpg/tool"

func (t *MyTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
    // Type-safe variable access
    userID, ok := tool.GetVariable[string](ctx, "user_id")

    // With default
    maxItems := tool.GetVariableOr[int](ctx, "max_items", 10)

    // Panic if missing (use sparingly)
    tenantID := tool.MustGetVariable[string](ctx, "tenant_id")

    // All variables
    vars := tool.GetVariables(ctx)

    // Full run context
    rc, ok := tool.GetRunContext(ctx)
    // rc.RunID, rc.SessionID, rc.Variables

    // Individual IDs
    runID, _ := tool.GetRunID(ctx)
    sessionID, _ := tool.GetSessionID(ctx)

    return "ok", nil
}
```

Variables are automatically propagated to child runs in agent-as-tool hierarchies. Instruction overrides (`OverrideInstructions`, `AppendInstructions`) are NOT propagated — each child agent uses its own system prompt.

## Tool Error Types

```go
import "github.com/youssefsiam38/agentpg/tool"

func (t *MyTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
    // Cancel: no retry, unrecoverable (bad auth, permission denied)
    return "", tool.ToolCancel(errors.New("unauthorized"))

    // Discard: permanent failure, invalid input
    return "", tool.ToolDiscard(errors.New("invalid parameters"))

    // Snooze: retry after duration WITHOUT consuming an attempt (rate limits)
    return "", tool.ToolSnooze(30*time.Second, errors.New("rate limited"))

    // Regular error: retried up to MaxAttempts with instant retry (default)
    return "", errors.New("temporary failure")
}
```

### Retry Configuration

```go
client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
    ToolRetryConfig: &agentpg.ToolRetryConfig{
        MaxAttempts: 2,    // Default: 2 (1 original + 1 retry)
        Jitter:      0.0,  // Default: 0 = instant retry
    },
})
```

## Database-Aware Tool

```go
type OrderTool struct {
    db *pgxpool.Pool
}

func (t *OrderTool) Name() string        { return "get_order" }
func (t *OrderTool) Description() string { return "Get order details by ID" }

func (t *OrderTool) InputSchema() tool.ToolSchema {
    return tool.ToolSchema{
        Type: "object",
        Properties: map[string]tool.PropertyDef{
            "order_id": {Type: "string", Description: "Order ID"},
        },
        Required: []string{"order_id"},
    }
}

func (t *OrderTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
    var p struct{ OrderID string `json:"order_id"` }
    if err := json.Unmarshal(input, &p); err != nil {
        return "", fmt.Errorf("invalid input: %w", err)
    }

    // Use run variables for tenant isolation
    tenantID, _ := tool.GetVariable[string](ctx, "tenant_id")

    var status, total string
    err := t.db.QueryRow(ctx,
        "SELECT status, total FROM orders WHERE id = $1 AND tenant_id = $2",
        p.OrderID, tenantID,
    ).Scan(&status, &total)
    if err != nil {
        return "", fmt.Errorf("order not found: %w", err)
    }

    return fmt.Sprintf("Order %s: status=%s, total=%s", p.OrderID, status, total), nil
}
```

## Registration and Agent Binding

```go
// 1. Register tools on client BEFORE Start
client.RegisterTool(&WeatherTool{apiKey: key})
client.RegisterTool(&OrderTool{db: pool})

// 2. Start client
client.Start(ctx)

// 3. Create agents with allowed tools AFTER Start
agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:  "assistant",
    Model: "claude-sonnet-4-5-20250929",
    Tools: []string{"get_weather", "get_order"},  // Must match registered tool names
})
```

An agent can only use tools that are BOTH registered on the client AND listed in the agent's `Tools` array. Tools not in the agent's list are invisible to it even if registered.
