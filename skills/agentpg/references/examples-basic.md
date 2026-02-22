# Examples: Basic, database/sql, Distributed

## Table of Contents

- [basic/01_simple_chat](#basic01_simple_chat) - Minimal AgentPG app
- [basic/02_shared_tools](#basic02_shared_tools) - Multiple agents sharing tools
- [database_sql](#database_sql) - Standard library driver + transactions
- [distributed](#distributed) - Multi-instance with leader election

---

## basic/01_simple_chat

Minimal AgentPG: database-driven agent, session, RunSync.

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

	apiKey := os.Getenv("ANTHROPIC_API_KEY")
	if apiKey == "" {
		log.Fatal("ANTHROPIC_API_KEY environment variable is required")
	}

	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		log.Fatal("DATABASE_URL environment variable is required")
	}

	pool, err := pgxpool.New(ctx, dbURL)
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer pool.Close()

	drv := pgxv5.New(pool)

	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: apiKey,
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	fmt.Printf("AgentPG client started (instance: %s)\n\n", client.InstanceID())

	agent, err := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "assistant",
		Description:  "A helpful coding assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful coding assistant. Be concise and accurate.",
	})
	if err != nil {
		log.Fatalf("Failed to get or create agent: %v", err)
	}

	fmt.Printf("Agent ready: %s (ID: %s)\n\n", agent.Name, agent.ID)

	sessionID, err := client.NewSession(ctx, nil, map[string]any{
		"tenant_id":   "tenant-1",
		"user_id":     "example-user",
		"description": "Basic example session",
	})
	if err != nil {
		log.Fatalf("Failed to create session: %v", err)
	}

	fmt.Printf("Created session: %s\n\n", sessionID)

	response, err := client.RunSync(ctx, sessionID, agent.ID, "Explain what the AgentPG package does in 3 sentences.", nil)
	if err != nil {
		log.Fatalf("Failed to run agent: %v", err)
	}

	fmt.Println("Agent response:")
	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	fmt.Printf("\nTokens used: %d input, %d output\n",
		response.Usage.InputTokens,
		response.Usage.OutputTokens)
	fmt.Printf("Stop reason: %s\n", response.StopReason)
}
```

---

## basic/02_shared_tools

Multiple agents sharing the same tools with different subsets.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type GetTimeTool struct{}

func (t *GetTimeTool) Name() string        { return "get_time" }
func (t *GetTimeTool) Description() string { return "Get the current time" }
func (t *GetTimeTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{Type: "object", Properties: map[string]tool.PropertyDef{}}
}
func (t *GetTimeTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return time.Now().Format(time.RFC3339), nil
}

type CalculatorTool struct{}

func (t *CalculatorTool) Name() string        { return "calculator" }
func (t *CalculatorTool) Description() string { return "Perform basic math operations" }
func (t *CalculatorTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"operation": {Type: "string", Enum: []string{"add", "subtract", "multiply", "divide"}},
			"a":         {Type: "number", Description: "First number"},
			"b":         {Type: "number", Description: "Second number"},
		},
		Required: []string{"operation", "a", "b"},
	}
}
func (t *CalculatorTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Operation string  `json:"operation"`
		A         float64 `json:"a"`
		B         float64 `json:"b"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", err
	}
	var result float64
	switch params.Operation {
	case "add":
		result = params.A + params.B
	case "subtract":
		result = params.A - params.B
	case "multiply":
		result = params.A * params.B
	case "divide":
		if params.B == 0 {
			return "", fmt.Errorf("division by zero")
		}
		result = params.A / params.B
	}
	return fmt.Sprintf("%.2f", result), nil
}

type WeatherTool struct{}

func (t *WeatherTool) Name() string        { return "get_weather" }
func (t *WeatherTool) Description() string { return "Get weather for a city (simulated)" }
func (t *WeatherTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"city": {Type: "string", Description: "City name"},
		},
		Required: []string{"city"},
	}
}
func (t *WeatherTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		City string `json:"city"`
	}
	json.Unmarshal(input, &params)
	return fmt.Sprintf("Weather in %s: 22C, Partly Cloudy", params.City), nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	apiKey := os.Getenv("ANTHROPIC_API_KEY")
	if apiKey == "" {
		log.Fatal("ANTHROPIC_API_KEY environment variable is required")
	}
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		log.Fatal("DATABASE_URL environment variable is required")
	}

	pool, err := pgxpool.New(ctx, dbURL)
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer pool.Close()

	drv := pgxv5.New(pool)

	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: apiKey,
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	// Register tools on the client (shared by all agents)
	client.RegisterTool(&GetTimeTool{})
	client.RegisterTool(&CalculatorTool{})
	client.RegisterTool(&WeatherTool{})

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	maxTokens := 1024

	// Agent 1: General Assistant - has access to all tools
	generalAssistant, err := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "general-assistant",
		Description:  "General purpose assistant with all tools",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful general assistant. Be concise.",
		Tools:        []string{"get_time", "calculator", "get_weather"},
		MaxTokens:    &maxTokens,
	})
	if err != nil {
		log.Fatalf("Failed to create general-assistant: %v", err)
	}

	// Agent 2: Math Tutor - only calculator
	mathTutor, err := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "math-tutor",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a math tutor. Help with calculations. Be concise.",
		Tools:        []string{"calculator"},
		MaxTokens:    &maxTokens,
	})
	if err != nil {
		log.Fatalf("Failed to create math-tutor: %v", err)
	}

	// Agent 3: Weather Bot - only weather
	weatherBot, err := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "weather-bot",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a weather assistant. Provide weather info. Be concise.",
		Tools:        []string{"get_weather"},
		MaxTokens:    &maxTokens,
	})
	if err != nil {
		log.Fatalf("Failed to create weather-bot: %v", err)
	}

	// Run each agent
	sid1, _ := client.NewSession(ctx, nil, nil)
	resp1, _ := client.RunSync(ctx, sid1, generalAssistant.ID,
		"What time is it, what's 15*7, and what's the weather in Paris?", nil)
	printResponse(resp1)

	sid2, _ := client.NewSession(ctx, nil, nil)
	resp2, _ := client.RunSync(ctx, sid2, mathTutor.ID, "What's 144 divided by 12?", nil)
	printResponse(resp2)

	sid3, _ := client.NewSession(ctx, nil, nil)
	resp3, _ := client.RunSync(ctx, sid3, weatherBot.ID, "What's the weather in Tokyo?", nil)
	printResponse(resp3)
}

func printResponse(resp *agentpg.Response) {
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			text := block.Text
			if len(text) > 300 {
				text = text[:300] + "..."
			}
			fmt.Printf("Response: %s\n", text)
		}
	}
	fmt.Printf("Tokens: %d in, %d out\n\n", resp.Usage.InputTokens, resp.Usage.OutputTokens)
}
```

---

## database_sql

Using standard library `database/sql` driver with transaction support.

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	_ "github.com/lib/pq"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/databasesql"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	apiKey := os.Getenv("ANTHROPIC_API_KEY")
	if apiKey == "" {
		log.Fatal("ANTHROPIC_API_KEY environment variable is required")
	}
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		log.Fatal("DATABASE_URL environment variable is required")
	}

	db, err := sql.Open("postgres", dbURL)
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer db.Close()

	if err := db.Ping(); err != nil {
		log.Fatalf("Failed to ping database: %v", err)
	}

	// database/sql driver requires connection string for LISTEN/NOTIFY
	drv := databasesql.New(db, dbURL)

	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: apiKey,
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}

	maxTokens := 1024
	agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "sql-demo-agent",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Keep responses concise.",
		MaxTokens:    &maxTokens,
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, map[string]any{"driver": "database/sql"})

	// Basic conversation
	prompts := []string{
		"Hello! What's the capital of Japan?",
		"What about France?",
		"Thanks! What were my previous questions about?",
	}
	for i, prompt := range prompts {
		fmt.Printf("=== Message %d ===\nUser: %s\n", i+1, prompt)
		response, err := client.RunFastSync(ctx, sessionID, agent.ID, prompt, nil)
		if err != nil {
			log.Printf("Error: %v\n\n", err)
			continue
		}
		for _, block := range response.Message.Content {
			if block.Type == agentpg.ContentTypeText {
				fmt.Printf("Agent: %s\n", block.Text)
			}
		}
		fmt.Printf("Tokens: %d input, %d output\n\n",
			response.Usage.InputTokens, response.Usage.OutputTokens)
	}

	// Transaction example: NewSessionTx + RunFastTx
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		log.Fatalf("Failed to begin transaction: %v", err)
	}

	sessionID2, err := client.NewSessionTx(ctx, tx, nil, map[string]any{
		"description": "Transaction demo session",
	})
	if err != nil {
		tx.Rollback()
		log.Fatalf("Failed to create session in transaction: %v", err)
	}

	runID, err := client.RunFastTx(ctx, tx, sessionID2, agent.ID, "What's a fun fact about Tokyo?", nil)
	if err != nil {
		tx.Rollback()
		log.Fatalf("Failed to create run in transaction: %v", err)
	}

	// Commit FIRST, then wait
	if err := tx.Commit(); err != nil {
		log.Fatalf("Failed to commit transaction: %v", err)
	}

	response, err := client.WaitForRun(ctx, runID)
	if err != nil {
		log.Fatalf("Failed to wait for run: %v", err)
	}
	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Printf("Agent: %s\n", block.Text)
		}
	}
}
```

---

## distributed

Multi-instance deployment with leader election and calculator tool.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type CalculatorTool struct{}

func (t *CalculatorTool) Name() string { return "calculator" }
func (t *CalculatorTool) Description() string {
	return "Perform basic arithmetic operations (add, subtract, multiply, divide)"
}
func (t *CalculatorTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"operation": {Type: "string", Enum: []string{"add", "subtract", "multiply", "divide"}},
			"a":         {Type: "number", Description: "First operand"},
			"b":         {Type: "number", Description: "Second operand"},
		},
		Required: []string{"operation", "a", "b"},
	}
}
func (t *CalculatorTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Operation string  `json:"operation"`
		A         float64 `json:"a"`
		B         float64 `json:"b"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}
	var result float64
	switch params.Operation {
	case "add":
		result = params.A + params.B
	case "subtract":
		result = params.A - params.B
	case "multiply":
		result = params.A * params.B
	case "divide":
		if params.B == 0 {
			return "", fmt.Errorf("division by zero")
		}
		result = params.A / params.B
	}
	return fmt.Sprintf("%.2f", result), nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	apiKey := os.Getenv("ANTHROPIC_API_KEY")
	if apiKey == "" {
		log.Fatal("ANTHROPIC_API_KEY environment variable is required")
	}
	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		log.Fatal("DATABASE_URL environment variable is required")
	}

	pool, err := pgxpool.New(ctx, dbURL)
	if err != nil {
		log.Fatalf("Failed to connect to database: %v", err)
	}
	defer pool.Close()

	drv := pgxv5.New(pool)

	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey:             apiKey,
		MaxConcurrentRuns:  10,
		MaxConcurrentTools: 50,
		BatchPollInterval:  30 * time.Second,
		RunPollInterval:    1 * time.Second,
		ToolPollInterval:   500 * time.Millisecond,
		HeartbeatInterval:  15 * time.Second,
		LeaderTTL:          30 * time.Second,
		StuckRunTimeout:    5 * time.Minute,
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	client.RegisterTool(&CalculatorTool{})

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	log.Printf("Client started (instance ID: %s)", client.InstanceID())

	chatAgent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "chat",
		Description:  "General purpose chat agent with math capabilities",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Use the calculator tool when asked to perform math operations.",
		Tools:        []string{"calculator"},
	})
	if err != nil {
		log.Fatalf("Failed to create chat agent: %v", err)
	}

	sessionID, _ := client.NewSession(ctx, nil, map[string]any{
		"tenant_id": "tenant1",
		"user_id":   "user-123",
	})

	response, err := client.RunSync(ctx, sessionID, chatAgent.ID, "What is 42 * 17?", nil)
	if err != nil {
		log.Fatalf("Failed to run agent: %v", err)
	}

	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	// Continue the conversation
	response, _ = client.RunSync(ctx, sessionID, chatAgent.ID, "Now divide that result by 7", nil)
	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```
