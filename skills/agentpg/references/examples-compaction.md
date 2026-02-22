# Examples: Context Compaction & Extended Context

## Table of Contents

- [01_auto_compaction](#01_auto_compaction) - Auto compaction via ClientConfig
- [02_manual_compaction](#02_manual_compaction) - Manual Compact() with before/after comparison
- [03_custom_strategy](#03_custom_strategy) - Hybrid vs Summarization strategies
- [04_compaction_monitoring](#04_compaction_monitoring) - CompactionMonitor and TokenTracker
- [extended_context](#extended_context) - 1M token extended context window

---

## 01_auto_compaction

Auto compaction enabled via `ClientConfig.AutoCompactionEnabled`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/compaction"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)

	// Enable auto-compaction in client config
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey:                os.Getenv("ANTHROPIC_API_KEY"),
		AutoCompactionEnabled: true,
		CompactionConfig: &compaction.Config{
			Strategy:        compaction.StrategyHybrid, // Prune tool outputs first, then summarize
			Trigger:         0.85,                      // Trigger at 85% context usage
			TargetTokens:    80000,                     // Target after compaction
			PreserveLastN:   10,                        // Always keep last 10 messages
			ProtectedTokens: 40000,                     // Never touch last 40K tokens
		},
	})

	client.Start(ctx)

	maxTokens := 4096
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "compaction-demo",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Provide detailed, thorough responses.",
		MaxTokens:    &maxTokens,
	})
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, map[string]any{
		"description": "Auto compaction demonstration",
	})

	// Long conversation that may trigger compaction
	questions := []string{
		"Explain the history of computer programming from the 1950s to today.",
		"Compare object-oriented programming with functional programming.",
		"Describe how databases have evolved from hierarchical models to distributed systems.",
		"Explain clean code principles and software architecture.",
		"Discuss the evolution of web development from static HTML to modern frameworks.",
	}

	for i, question := range questions {
		fmt.Printf("=== Question %d/%d ===\n", i+1, len(questions))

		response, _ := client.RunFastSync(ctx, sessionID, agent.ID, question, nil)
		fmt.Printf("Tokens - Input: %d, Output: %d\n",
			response.Usage.InputTokens, response.Usage.OutputTokens)

		// Check compaction stats after each run
		stats, _ := client.GetCompactionStats(ctx, sessionID)
		fmt.Printf("Context Usage: %.1f%% (%d tokens)\n", stats.UsagePercent*100, stats.TotalTokens)
		fmt.Printf("Messages: %d total, %d compactable\n",
			stats.TotalMessages, stats.CompactableMessages)
		if stats.CompactionCount > 0 {
			fmt.Printf("Compaction Events: %d\n", stats.CompactionCount)
		}
		fmt.Println()
	}
}
```

---

## 02_manual_compaction

Manual compaction with `Compact()`, `CompactIfNeeded()`, `NeedsCompaction()`, before/after comparison.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"strings"
	"syscall"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/compaction"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

// VerboseSearchTool generates large outputs to fill context quickly
type VerboseSearchTool struct{}

func (v *VerboseSearchTool) Name() string        { return "search" }
func (v *VerboseSearchTool) Description() string { return "Search for information (returns verbose results)" }
func (v *VerboseSearchTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"query": {Type: "string", Description: "Search query"},
		},
		Required: []string{"query"},
	}
}
func (v *VerboseSearchTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct{ Query string `json:"query"` }
	json.Unmarshal(input, &params)

	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("Search results for '%s':\n\n", params.Query))
	for i := 1; i <= 5; i++ {
		sb.WriteString(fmt.Sprintf("## Result %d: %s - Comprehensive Guide\n", i, params.Query))
		sb.WriteString("This comprehensive article covers the topic in great detail. ")
		sb.WriteString("It includes background information, best practices, examples, and common pitfalls.\n\n")
	}
	return sb.String(), nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)

	// Auto-compaction DISABLED for manual control
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey:                os.Getenv("ANTHROPIC_API_KEY"),
		AutoCompactionEnabled: false,
		CompactionConfig: &compaction.Config{
			Strategy:     compaction.StrategyHybrid,
			Trigger:      0.50, // Lower threshold for demo
			TargetTokens: 5000,
		},
	})

	client.RegisterTool(&VerboseSearchTool{})
	client.Start(ctx)

	maxTokens := 1024
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "research-assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a research assistant. Use the search tool. Keep responses brief.",
		MaxTokens:    &maxTokens,
		Tools:        []string{"search"},
	})
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Build up context with queries
	queries := []string{
		"Search for microservices architecture",
		"Search for API design patterns",
		"Search for database optimization",
		"Search for Docker containerization",
	}
	for _, query := range queries {
		client.RunFastSync(ctx, sessionID, agent.ID, query, nil)
	}

	// Stats BEFORE compaction
	statsBefore, _ := client.GetCompactionStats(ctx, sessionID)
	fmt.Printf("Before: %d tokens, %d messages\n", statsBefore.TotalTokens, statsBefore.TotalMessages)

	// Manual compaction
	needsCompaction, _ := client.NeedsCompaction(ctx, sessionID)
	if needsCompaction {
		result, _ := client.Compact(ctx, sessionID)
		fmt.Printf("Compaction: %d -> %d tokens (%.1f%% reduction)\n",
			result.OriginalTokens, result.CompactedTokens,
			100.0*(1.0-float64(result.CompactedTokens)/float64(result.OriginalTokens)))
		fmt.Printf("Messages removed: %d, Summary created: %v\n",
			result.MessagesRemoved, result.SummaryCreated)
	} else {
		// CompactIfNeeded only compacts if threshold reached
		result, _ := client.CompactIfNeeded(ctx, sessionID)
		if result == nil {
			fmt.Println("No compaction performed (threshold not reached)")
		}
	}

	// Stats AFTER compaction
	statsAfter, _ := client.GetCompactionStats(ctx, sessionID)
	fmt.Printf("After: %d tokens, %d messages\n", statsAfter.TotalTokens, statsAfter.TotalMessages)

	// Verify conversation continues
	response, _ := client.RunFastSync(ctx, sessionID, agent.ID,
		"Based on our previous discussion, what topics did we cover?", nil)
	fmt.Println(response.Text)
}
```

---

## 03_custom_strategy

Comparing Hybrid vs Summarization strategies using `CompactWithConfig`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/compaction"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
)

func main() {
	ctx := context.Background()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)

	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey:                os.Getenv("ANTHROPIC_API_KEY"),
		AutoCompactionEnabled: false,
		CompactionConfig: &compaction.Config{
			Strategy:     compaction.StrategyHybrid,
			Trigger:      0.30,
			TargetTokens: 3000,
		},
	})

	// ... register tools, start client, create agent ...
	client.Start(ctx)
	defer client.Stop(context.Background())

	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "doc-assistant", Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a documentation assistant.",
	})

	// Demo 1: Default Hybrid strategy
	sessionID1, _ := client.NewSession(ctx, nil, nil)
	runConversation(ctx, client, sessionID1, agent.ID)

	// Compact with default (Hybrid) config
	result1, _ := client.Compact(ctx, sessionID1)
	fmt.Printf("Hybrid: %d -> %d tokens\n", result1.OriginalTokens, result1.CompactedTokens)

	// Demo 2: Override with Summarization strategy using CompactWithConfig
	sessionID2, _ := client.NewSession(ctx, nil, nil)
	runConversation(ctx, client, sessionID2, agent.ID)

	result2, _ := client.CompactWithConfig(ctx, sessionID2, &compaction.Config{
		Strategy:     compaction.StrategySummarization, // Direct summarization
		Trigger:      0.30,
		TargetTokens: 3000,
	})
	fmt.Printf("Summarization: %d -> %d tokens\n", result2.OriginalTokens, result2.CompactedTokens)
}

func runConversation(ctx context.Context, client *agentpg.Client[pgx.Tx], sessionID, agentID uuid.UUID) {
	topics := []string{
		"Get documentation for Kubernetes deployment",
		"Get documentation for Docker networking",
		"Get documentation for PostgreSQL indexing",
	}
	for _, topic := range topics {
		client.RunFastSync(ctx, sessionID, agentID, topic, nil)
	}
}
```

---

## 04_compaction_monitoring

CompactionMonitor and TokenTracker for tracking compaction events and metrics.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"sync"
	"time"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/compaction"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
)

type CompactionMonitor struct {
	mu     sync.Mutex
	events []CompactionEvent
}

type CompactionEvent struct {
	Timestamp       time.Time
	SessionID       uuid.UUID
	OriginalTokens  int
	CompactedTokens int
	Reduction       float64
}

func (m *CompactionMonitor) RecordEvent(sessionID uuid.UUID, result *compaction.Result) {
	m.mu.Lock()
	defer m.mu.Unlock()
	reduction := 100.0 * (1.0 - float64(result.CompactedTokens)/float64(result.OriginalTokens))
	m.events = append(m.events, CompactionEvent{
		Timestamp: time.Now(), SessionID: sessionID,
		OriginalTokens: result.OriginalTokens, CompactedTokens: result.CompactedTokens,
		Reduction: reduction,
	})
	fmt.Printf("[COMPACTION] %d -> %d tokens (%.1f%% reduction)\n",
		result.OriginalTokens, result.CompactedTokens, reduction)
}

type TokenTracker struct {
	mu      sync.Mutex
	samples []TokenSample
}

type TokenSample struct {
	Timestamp   time.Time
	TotalTokens int
	Messages    int
}

func (t *TokenTracker) RecordSample(stats *compaction.Stats) {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.samples = append(t.samples, TokenSample{
		Timestamp: time.Now(), TotalTokens: stats.TotalTokens, Messages: stats.TotalMessages,
	})
}

func main() {
	ctx := context.Background()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey:                os.Getenv("ANTHROPIC_API_KEY"),
		AutoCompactionEnabled: false,
		CompactionConfig: &compaction.Config{
			Strategy: compaction.StrategyHybrid, Trigger: 0.30, TargetTokens: 3000,
		},
	})

	client.Start(ctx)
	maxTokens := 4096
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "monitored-assistant", Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful assistant. Provide detailed responses.",
		MaxTokens:    &maxTokens,
	})
	defer client.Stop(context.Background())

	monitor := &CompactionMonitor{}
	tracker := &TokenTracker{}

	sessionID, _ := client.NewSession(ctx, nil, nil)

	prompts := []string{
		"Explain the complete history of artificial intelligence.",
		"Describe all types of neural network architectures.",
		"Explain the mathematics behind backpropagation.",
		"Discuss ethical considerations of AI systems.",
		"Overview of natural language processing and LLMs.",
	}

	for i, prompt := range prompts {
		fmt.Printf("--- Query %d/%d ---\n", i+1, len(prompts))
		response, _ := client.RunFastSync(ctx, sessionID, agent.ID, prompt, nil)
		fmt.Printf("Tokens - Input: %d, Output: %d\n",
			response.Usage.InputTokens, response.Usage.OutputTokens)

		stats, _ := client.GetCompactionStats(ctx, sessionID)
		tracker.RecordSample(stats)
		fmt.Printf("Context: %d tokens (%.1f%%)\n", stats.TotalTokens, stats.UsagePercent*100)

		if stats.NeedsCompaction {
			result, err := client.Compact(ctx, sessionID)
			if err != nil {
				log.Printf("Compaction failed: %v", err)
			} else {
				monitor.RecordEvent(sessionID, result)
			}
		}
	}

	// Database monitoring queries
	fmt.Println("\nDatabase queries:")
	fmt.Printf("  SELECT * FROM agentpg_compaction_events WHERE session_id = '%s';\n", sessionID)
	fmt.Printf("  SELECT * FROM agentpg_message_archive WHERE session_id = '%s';\n", sessionID)
}
```

---

## extended_context

Extended context window (1M tokens) via `AgentDefinition.Config`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strings"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
)

func generateLongDocument(sections int) string {
	var sb strings.Builder
	sb.WriteString("# Comprehensive Technical Documentation\n\n")
	topics := []string{"System Architecture", "Database Design", "API Specifications",
		"Security Guidelines", "Performance Optimization"}
	for i := 0; i < sections; i++ {
		topic := topics[i%len(topics)]
		sb.WriteString(fmt.Sprintf("## Section %d: %s\n\n", i+1, topic))
		for j := 0; j < 5; j++ {
			sb.WriteString(fmt.Sprintf("### %s - Part %d\n\n", topic, j+1))
			sb.WriteString(fmt.Sprintf("This section covers important aspects of %s. ", topic))
			sb.WriteString("Understanding these concepts is essential for building robust systems.\n\n")
		}
	}
	return sb.String()
}

func main() {
	ctx := context.Background()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"),
	})

	client.Start(ctx)
	defer client.Stop(context.Background())

	maxTokens := 4096
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "extended-context-demo",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a document analysis assistant.",
		MaxTokens:    &maxTokens,
		Config: map[string]any{
			"extended_context": true,  // Enable 1M token support
			"auto_compaction":  false, // Disable compaction - rely on extended context
		},
	})

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Generate and process a long document
	document := generateLongDocument(20)
	fmt.Printf("Generated document: %d characters (~%d tokens estimated)\n", len(document), len(document)/4)

	prompt := fmt.Sprintf("Analyze this document and summarize its structure:\n\n%s", document)
	response, err := client.RunFastSync(ctx, sessionID, agent.ID, prompt, nil)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("Response: %s\n", response.Text[:min(500, len(response.Text))])
	fmt.Printf("Tokens - Input: %d, Output: %d\n",
		response.Usage.InputTokens, response.Usage.OutputTokens)

	// Follow-up questions work with full context
	resp2, _ := client.RunFastSync(ctx, sessionID, agent.ID,
		"What security guidelines are mentioned in the document?", nil)
	fmt.Println(resp2.Text)
}

func min(a, b int) int {
	if a < b {
		return a
	}
	return b
}
```

**Extended context features:**
- Automatic fallback: If API returns max_tokens error, retries with extended context header
- Beta header injection: Adds `anthropic-beta: context-1m-2025-08-07`
- Simple config: Just add `extended_context: true` to AgentDefinition.Config
