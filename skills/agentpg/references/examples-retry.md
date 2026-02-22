# Retry & Rescue Examples

## Table of Contents
- [01 Instant Retry (Default)](#01-instant-retry) - Default retry with 80% failure rate tool
- [02 Error Types](#02-error-types) - ToolCancel, ToolDiscard, ToolSnooze, regular errors
- [03 Exponential Backoff](#03-exponential-backoff) - Opt-in backoff with Jitter > 0

---

## 01 Instant Retry

Default retry behavior: 2 attempts, instant retry (no delay). FlakyTool with 80% failure rate.

```go
// Package main demonstrates the default instant retry behavior.
//
// This example shows:
// - Default retry configuration (2 attempts, instant retry)
// - Tool that fails sometimes but succeeds on retry
// - Snappy user experience with no delay between retries
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand/v2"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

// FlakyTool simulates an unreliable external service.
// It fails randomly ~80% of the time, but the default instant retry
// ensures a snappy user experience.
type FlakyTool struct {
	callCount int
}

func (t *FlakyTool) Name() string {
	return "flaky_service"
}

func (t *FlakyTool) Description() string {
	return "Simulates an unreliable external service that may fail occasionally"
}

func (t *FlakyTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"query": {
				Type:        "string",
				Description: "The query to send to the service",
			},
		},
		Required: []string{"query"},
	}
}

func (t *FlakyTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	t.callCount++
	var params struct {
		Query string `json:"query"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}

	// Simulate 80% failure rate
	if rand.Float64() < 0.8 {
		log.Printf("[FlakyTool] Call #%d FAILED (query: %q)", t.callCount, params.Query)
		return "", fmt.Errorf("temporary service error")
	}

	log.Printf("[FlakyTool] Call #%d SUCCESS (query: %q)", t.callCount, params.Query)
	return fmt.Sprintf("Result for query '%s': Service responded successfully!", params.Query), nil
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

	// Create client with DEFAULT retry config (instant retry)
	// This is the snappy default:
	// - MaxAttempts: 2 (1 retry on failure)
	// - Jitter: 0.0 (instant retry, no delay)
	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: apiKey,
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	// Register the flaky tool
	flakyTool := &FlakyTool{}
	if err := client.RegisterTool(flakyTool); err != nil {
		log.Fatalf("Failed to register tool: %v", err)
	}

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	// Create agent with access to the flaky tool in the database
	agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Use the flaky_service tool to answer user queries.",
		Tools:        []string{"flaky_service"},
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}

	log.Println("Client started with DEFAULT instant retry (2 attempts, no delay)")
	log.Println("")

	// Create session and run
	sessionID, err := client.NewSession(ctx, nil, map[string]any{"demo": "retry"})
	if err != nil {
		log.Fatalf("Failed to create session: %v", err)
	}

	log.Println("--- Sending request that uses the flaky tool ---")
	log.Println("(The tool has 80% failure rate, but instant retry makes it snappy)")
	log.Println("")

	start := time.Now()
	response, err := client.RunFastSync(ctx, sessionID, agent.ID, "Query the service for 'weather forecast' once, never retry, if it fails. if it succeeds, provide the result.", nil)
	elapsed := time.Since(start)

	if err != nil {
		log.Printf("Request failed: %v", err)
	} else {
		fmt.Println("\nAgent response:")
		for _, block := range response.Message.Content {
			if block.Type == agentpg.ContentTypeText {
				fmt.Println(block.Text)
			}
		}
	}

	fmt.Printf("\nTotal tool calls: %d\n", flakyTool.callCount)
	fmt.Printf("Total time: %v\n", elapsed)
	fmt.Println("\nNote: With instant retry, failures are recovered immediately!")
}
```

---

## 02 Error Types

Demonstrates ToolCancel, ToolDiscard, ToolSnooze, and regular error retry behavior.

```go
// Package main demonstrates the different tool error types.
//
// This example shows:
// - ToolCancel: Immediate cancellation, no retry (e.g., auth failure)
// - ToolDiscard: Permanent failure, invalid input (e.g., bad parameters)
// - ToolSnooze: Retry after duration without consuming attempt (e.g., rate limit)
// - Regular errors: Consumed retry attempts with instant retry
package main

import (
	"context"
	"encoding/json"
	"errors"
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

// APITool demonstrates different error types based on input.
type APITool struct {
	callCount int
}

func (t *APITool) Name() string {
	return "api_call"
}

func (t *APITool) Description() string {
	return "Makes an API call. Use action='auth_fail' to simulate auth failure, 'invalid' for bad input, 'rate_limit' for rate limiting, 'flaky' for random failures, or 'success' for normal operation."
}

func (t *APITool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"action": {
				Type:        "string",
				Description: "The action to perform",
				Enum:        []string{"auth_fail", "invalid", "rate_limit", "flaky", "success"},
			},
		},
		Required: []string{"action"},
	}
}

func (t *APITool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	t.callCount++
	var params struct {
		Action string `json:"action"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}

	log.Printf("[APITool] Call #%d - action: %s", t.callCount, params.Action)

	switch params.Action {
	case "auth_fail":
		// ToolCancel: Authentication failed - never retry
		// Use when the error is unrecoverable (bad API key, permission denied)
		log.Printf("[APITool] Returning ToolCancel (no retry)")
		return "", tool.ToolCancel(errors.New("authentication failed: invalid API key"))

	case "invalid":
		// ToolDiscard: Invalid input - never retry
		// Use when the input is fundamentally wrong (malformed data, impossible request)
		log.Printf("[APITool] Returning ToolDiscard (no retry)")
		return "", tool.ToolDiscard(errors.New("invalid action requested"))

	case "rate_limit":
		// ToolSnooze: Rate limited - retry after duration WITHOUT consuming attempt
		// Use for temporary unavailability (rate limits, service maintenance)
		log.Printf("[APITool] Returning ToolSnooze (retry after 5s, does NOT consume attempt)")
		return "", tool.ToolSnooze(5*time.Second, errors.New("rate limit exceeded"))

	case "flaky":
		// Regular error: Will be retried (consumes an attempt)
		// Default behavior: instant retry up to MaxAttempts
		if t.callCount < 3 {
			log.Printf("[APITool] Returning regular error (will retry instantly)")
			return "", errors.New("temporary network error")
		}
		log.Printf("[APITool] Finally succeeded after retries!")
		return "Flaky service finally responded!", nil

	case "success":
		log.Printf("[APITool] Success!")
		return "API call successful!", nil

	default:
		return "", fmt.Errorf("unknown action: %s", params.Action)
	}
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
		// Use default retry config (2 attempts, instant retry)
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	apiTool := &APITool{}
	if err := client.RegisterTool(apiTool); err != nil {
		log.Fatalf("Failed to register tool: %v", err)
	}

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	// Create agent in the database
	agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Use the api_call tool to perform API operations.",
		Tools:        []string{"api_call"},
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}

	log.Println("=== Tool Error Types Demo ===")
	log.Println("")
	log.Println("Error Types:")
	log.Println("  ToolCancel  - Immediate failure, no retry (e.g., auth failure)")
	log.Println("  ToolDiscard - Permanent failure, invalid input")
	log.Println("  ToolSnooze  - Retry after delay, does NOT consume attempt")
	log.Println("  Regular err - Retried instantly up to MaxAttempts")
	log.Println("")

	// Demo each error type
	demos := []struct {
		name   string
		prompt string
	}{
		{
			name:   "SUCCESS",
			prompt: "Call the API with action 'success'",
		},
		{
			name:   "AUTH FAILURE (ToolCancel)",
			prompt: "Call the API with action 'auth_fail'",
		},
		{
			name:   "INVALID INPUT (ToolDiscard)",
			prompt: "Call the API with action 'invalid'",
		},
		// Note: rate_limit and flaky demos would take longer due to snooze/retry
		// Uncomment to test:
		// {
		// 	name:   "RATE LIMIT (ToolSnooze)",
		// 	prompt: "Call the API with action 'rate_limit'",
		// },
	}

	for i, demo := range demos {
		sessionID, err := client.NewSession(ctx, nil, map[string]any{"demo": fmt.Sprintf("error-types-%d", i)})
		if err != nil {
			log.Fatalf("Failed to create session: %v", err)
		}

		log.Printf("\n--- Demo: %s ---\n", demo.name)
		apiTool.callCount = 0

		response, err := client.RunFastSync(ctx, sessionID, agent.ID, demo.prompt, nil)
		if err != nil {
			log.Printf("Error: %v", err)
		} else {
			fmt.Println("Agent response:")
			for _, block := range response.Message.Content {
				if block.Type == agentpg.ContentTypeText {
					fmt.Println(block.Text)
				}
			}
		}
		fmt.Printf("Tool calls made: %d\n", apiTool.callCount)
	}

	log.Println("\n=== Demo Complete ===")
	log.Println("")
	log.Println("Key takeaways:")
	log.Println("- ToolCancel/ToolDiscard: Fail immediately, no retry")
	log.Println("- ToolSnooze: Retry after delay without consuming attempts")
	log.Println("- Regular errors: Instant retry (default) up to MaxAttempts")
}
```

---

## 03 Exponential Backoff

Opt-in exponential backoff with `Jitter > 0`. Uses River's `attempt^4` formula.

```go
// Package main demonstrates opt-in exponential backoff for retries.
//
// This example shows:
// - Configuring exponential backoff with Jitter > 0
// - River's attempt^4 formula (1s, 16s, 81s, 256s, 625s...)
// - When to use backoff (external APIs with rate limits)
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

// RateLimitedAPITool simulates an external API with strict rate limits.
// With exponential backoff, failed requests wait progressively longer,
// giving the external service time to recover.
type RateLimitedAPITool struct {
	callCount     int64
	lastCallTime  time.Time
	failuresUntil int
}

func (t *RateLimitedAPITool) Name() string {
	return "external_api"
}

func (t *RateLimitedAPITool) Description() string {
	return "Calls an external API with rate limiting"
}

func (t *RateLimitedAPITool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"endpoint": {
				Type:        "string",
				Description: "The API endpoint to call",
			},
		},
		Required: []string{"endpoint"},
	}
}

func (t *RateLimitedAPITool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	count := atomic.AddInt64(&t.callCount, 1)
	now := time.Now()

	var params struct {
		Endpoint string `json:"endpoint"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}

	// Calculate time since last call
	timeSinceLastCall := time.Duration(0)
	if !t.lastCallTime.IsZero() {
		timeSinceLastCall = now.Sub(t.lastCallTime)
	}
	t.lastCallTime = now

	log.Printf("[ExternalAPI] Call #%d to %s (time since last: %v)",
		count, params.Endpoint, timeSinceLastCall.Round(time.Millisecond))

	// Fail the first few calls to demonstrate backoff
	if int(count) <= t.failuresUntil {
		log.Printf("[ExternalAPI] Simulating rate limit error (will retry with backoff)")
		return "", fmt.Errorf("rate limit exceeded, please retry later")
	}

	log.Printf("[ExternalAPI] Success!")
	return fmt.Sprintf("API response from %s: OK", params.Endpoint), nil
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

	// Create client with EXPONENTIAL BACKOFF retry config
	// Set Jitter > 0 to enable River's attempt^4 backoff formula
	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: apiKey,

		// Opt-in to exponential backoff by setting Jitter > 0
		ToolRetryConfig: &agentpg.ToolRetryConfig{
			MaxAttempts: 7,   // More attempts for unreliable services
			Jitter:      0.1, // 10% jitter enables exponential backoff
		},
	})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	// This tool will fail the first 5 calls
	apiTool := &RateLimitedAPITool{failuresUntil: 5}
	if err := client.RegisterTool(apiTool); err != nil {
		log.Fatalf("Failed to register tool: %v", err)
	}

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}
	defer client.Stop(context.Background())

	// Create agent in the database
	agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Use the external_api tool to make API calls.",
		Tools:        []string{"external_api"},
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}

	log.Println("=== Exponential Backoff Demo ===")
	log.Println("")
	log.Println("Configuration:")
	log.Println("  MaxAttempts: 5")
	log.Println("  Jitter: 0.1 (enables backoff)")
	log.Println("")
	log.Println("Backoff delays (attempt^4 formula):")
	log.Println("  Attempt 1: ~1 second")
	log.Println("  Attempt 2: ~16 seconds")
	log.Println("  Attempt 3: ~81 seconds")
	log.Println("  Attempt 4: ~256 seconds")
	log.Println("  Attempt 5: ~625 seconds")
	log.Println("")
	log.Println("The tool will fail first 2 calls to show backoff in action...")
	log.Println("")

	sessionID, err := client.NewSession(ctx, nil, map[string]any{"demo": "backoff"})
	if err != nil {
		log.Fatalf("Failed to create session: %v", err)
	}

	log.Println("--- Starting request ---")
	start := time.Now()

	response, err := client.RunFastSync(ctx, sessionID, agent.ID, "Call the external API endpoint '/users'", nil)
	elapsed := time.Since(start)

	if err != nil {
		log.Printf("Error: %v", err)
	} else {
		fmt.Println("\nAgent response:")
		for _, block := range response.Message.Content {
			if block.Type == agentpg.ContentTypeText {
				fmt.Println(block.Text)
			}
		}
	}

	fmt.Printf("\nTotal calls: %d\n", atomic.LoadInt64(&apiTool.callCount))
	fmt.Printf("Total time: %v\n", elapsed.Round(time.Millisecond))

	log.Println("\n=== Demo Complete ===")
	log.Println("")
	log.Println("When to use exponential backoff:")
	log.Println("- External APIs with rate limits")
	log.Println("- Services that need recovery time")
	log.Println("- High-volume batch processing")
	log.Println("")
	log.Println("When to use instant retry (default):")
	log.Println("- Interactive applications")
	log.Println("- Tools with transient failures")
	log.Println("- When fast feedback is important")
}
```
