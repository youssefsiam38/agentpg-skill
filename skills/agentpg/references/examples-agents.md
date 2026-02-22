# Examples: Nested Agents (Agent-as-Tool)

## Table of Contents

- [01_basic_delegation](#01_basic_delegation) - Agent-as-tool with AgentIDs field
- [02_specialist_agents](#02_specialist_agents) - Multiple specialists with orchestrator
- [03_multi_level_hierarchy](#03_multi_level_hierarchy) - 3-level PM -> Lead -> Worker

---

## 01_basic_delegation

Basic agent-as-tool: research specialist delegated from orchestrator via `AgentIDs`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY")})

	client.Start(ctx)
	defer client.Stop(context.Background())

	maxTokens := 10000

	// Create child agent FIRST (research specialist)
	researchSpecialist, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:        "research-specialist",
		Description: "A research specialist for detailed analysis",
		Model:       "claude-sonnet-4-5-20250929",
		SystemPrompt: `You are a research specialist. Your role is to:
1. Analyze topics thoroughly
2. Provide detailed explanations with examples
3. Break down complex concepts into understandable parts`,
		MaxTokens: &maxTokens,
	})
	if err != nil {
		log.Fatal(err)
	}

	// Create parent orchestrator with delegation via AgentIDs
	orchestrator, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:        "orchestrator",
		Description: "Main assistant that can delegate to specialists",
		Model:       "claude-sonnet-4-5-20250929",
		SystemPrompt: `You are a helpful assistant that can delegate research tasks to a specialist.
When users ask for detailed information, use the research agent tool.
For simple questions, answer directly without delegation.`,
		AgentIDs:  []uuid.UUID{researchSpecialist.ID}, // research-specialist becomes a callable tool
		MaxTokens: &maxTokens,
	})
	if err != nil {
		log.Fatal(err)
	}

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Simple question - no delegation
	resp1, _ := client.RunSync(ctx, sessionID, orchestrator.ID, "What is 2 + 2?", nil)
	for _, block := range resp1.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	// Research question - triggers delegation to research-specialist
	resp2, _ := client.RunSync(ctx, sessionID, orchestrator.ID,
		"Can you research and explain how neural networks learn?", nil)
	for _, block := range resp2.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```

---

## 02_specialist_agents

Multiple specialists (code, data, research) with own tools, coordinated by orchestrator.

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

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type CodeAnalysisTool struct{}

func (c *CodeAnalysisTool) Name() string        { return "analyze_code" }
func (c *CodeAnalysisTool) Description() string { return "Analyze code for patterns and issues" }
func (c *CodeAnalysisTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"code":     {Type: "string", Description: "The code to analyze"},
			"language": {Type: "string", Enum: []string{"go", "python", "javascript", "typescript"}},
		},
		Required: []string{"code", "language"},
	}
}
func (c *CodeAnalysisTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Code     string `json:"code"`
		Language string `json:"language"`
	}
	json.Unmarshal(input, &params)
	lines := strings.Count(params.Code, "\n") + 1
	return fmt.Sprintf("Code Analysis (%s):\n- Lines: %d\n- Language: %s", params.Language, lines, params.Language), nil
}

type DataQueryTool struct{}

func (d *DataQueryTool) Name() string { return "query_data" }
func (d *DataQueryTool) Description() string {
	return "Query and analyze datasets (simulated)"
}
func (d *DataQueryTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"dataset":   {Type: "string", Enum: []string{"sales", "users", "products", "logs"}},
			"operation": {Type: "string", Enum: []string{"count", "average", "sum", "list"}},
		},
		Required: []string{"dataset", "operation"},
	}
}
func (d *DataQueryTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Dataset   string `json:"dataset"`
		Operation string `json:"operation"`
	}
	json.Unmarshal(input, &params)
	data := map[string]map[string]string{
		"sales": {"count": "Total records: 15,432", "sum": "Total revenue: $1,967,100"},
		"users": {"count": "Total users: 5,892", "average": "Average age: 34.2 years"},
	}
	if result, ok := data[params.Dataset][params.Operation]; ok {
		return result, nil
	}
	return "No data found", nil
}

type SearchTool struct{}

func (s *SearchTool) Name() string        { return "search_knowledge" }
func (s *SearchTool) Description() string { return "Search knowledge base for information" }
func (s *SearchTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"query":  {Type: "string", Description: "Search query"},
			"domain": {Type: "string", Enum: []string{"technology", "science", "business", "general"}},
		},
		Required: []string{"query"},
	}
}
func (s *SearchTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Query  string `json:"query"`
		Domain string `json:"domain"`
	}
	json.Unmarshal(input, &params)
	return fmt.Sprintf("Search results for '%s' in %s domain:\n1. Relevant article\n2. Documentation\n3. Tutorial",
		params.Query, params.Domain), nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY")})

	// Register all tools
	client.RegisterTool(&CodeAnalysisTool{})
	client.RegisterTool(&DataQueryTool{})
	client.RegisterTool(&SearchTool{})

	client.Start(ctx)
	defer client.Stop(context.Background())

	maxTokens := 2048

	// Specialist agents with their own tools
	codeSpecialist, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "code-specialist", Description: "Code analysis specialist",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a code analysis specialist. Use analyze_code to get metrics.",
		Tools: []string{"analyze_code"}, MaxTokens: &maxTokens,
	})

	dataSpecialist, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "data-specialist", Description: "Data analysis specialist",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a data analysis specialist. Query data before providing insights.",
		Tools: []string{"query_data"}, MaxTokens: &maxTokens,
	})

	researchSpecialist, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "research-specialist", Description: "Research and knowledge specialist",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a research specialist. Use search_knowledge to find information.",
		Tools: []string{"search_knowledge"}, MaxTokens: &maxTokens,
	})

	// Orchestrator delegates to specialists (no direct tools)
	orchestrator, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "orchestrator", Description: "Orchestrator that coordinates specialists",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: `You are an orchestrator. Delegate tasks to specialists:
- Code Agent: code analysis and reviews
- Data Agent: data analysis and queries
- Research Agent: general research`,
		AgentIDs:  []uuid.UUID{codeSpecialist.ID, dataSpecialist.ID, researchSpecialist.ID},
		MaxTokens: &maxTokens,
	})

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Code question -> delegates to code-specialist
	resp, _ := client.RunSync(ctx, sessionID, orchestrator.ID,
		"Analyze this Go code:\nfunc fibonacci(n int) int {\n    if n <= 1 { return n }\n    return fibonacci(n-1) + fibonacci(n-2)\n}", nil)
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	// Data question -> delegates to data-specialist
	resp, _ = client.RunSync(ctx, sessionID, orchestrator.ID,
		"What are our total sales and how many users do we have?", nil)
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```

---

## 03_multi_level_hierarchy

3-level hierarchy: Project Manager -> Team Leads -> Workers with specialized tools.

```
Architecture:
  Level 1 (Project Manager)
      |-- Engineering Lead (Level 2)
      |       |-- Frontend Developer (Level 3) [lint_frontend]
      |       |-- Backend Developer (Level 3)  [run_tests]
      |       +-- Database Specialist (Level 3) [run_migration]
      +-- Design Lead (Level 2)
              +-- UX Designer (Level 3)        [review_design]
```

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

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

// Level 3 tools
type FrontendLintTool struct{}
func (f *FrontendLintTool) Name() string        { return "lint_frontend" }
func (f *FrontendLintTool) Description() string { return "Lint frontend code for issues" }
func (f *FrontendLintTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{"code": {Type: "string"}},
		Required: []string{"code"},
	}
}
func (f *FrontendLintTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return "Frontend lint passed: No ESLint warnings. React best practices followed.", nil
}

type BackendTestTool struct{}
func (b *BackendTestTool) Name() string        { return "run_tests" }
func (b *BackendTestTool) Description() string { return "Run backend tests" }
func (b *BackendTestTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{"package": {Type: "string"}},
		Required: []string{"package"},
	}
}
func (b *BackendTestTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return "Tests passed: 45/45 tests passed. Coverage: 87%.", nil
}

type DatabaseMigrationTool struct{}
func (d *DatabaseMigrationTool) Name() string        { return "run_migration" }
func (d *DatabaseMigrationTool) Description() string { return "Run database migration" }
func (d *DatabaseMigrationTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{"migration": {Type: "string"}},
		Required: []string{"migration"},
	}
}
func (d *DatabaseMigrationTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return "Migration completed: Applied 3 pending migrations. Schema up to date.", nil
}

type DesignReviewTool struct{}
func (d *DesignReviewTool) Name() string        { return "review_design" }
func (d *DesignReviewTool) Description() string { return "Review UI/UX design" }
func (d *DesignReviewTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{"component": {Type: "string"}},
		Required: []string{"component"},
	}
}
func (d *DesignReviewTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return "Design review: Accessibility score 94/100. Color contrast meets WCAG 2.1 AA.", nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"), Name: "hierarchy-demo",
	})

	// Register all tools BEFORE Start
	client.RegisterTool(&FrontendLintTool{})
	client.RegisterTool(&BackendTestTool{})
	client.RegisterTool(&DatabaseMigrationTool{})
	client.RegisterTool(&DesignReviewTool{})

	client.Start(ctx)
	defer client.Stop(context.Background())

	maxW, maxL, maxM := 1024, 1536, 2048

	// Level 3: Workers with specialized tools
	frontendDev, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "frontend-developer", Description: "Frontend developer specialist for React/TypeScript",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a frontend developer. Use lint_frontend to check code quality.",
		Tools: []string{"lint_frontend"}, MaxTokens: &maxW,
	})
	backendDev, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "backend-developer", Description: "Backend developer specialist for Go/APIs",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a backend developer. Use run_tests to verify code.",
		Tools: []string{"run_tests"}, MaxTokens: &maxW,
	})
	dbSpecialist, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "database-specialist", Description: "Database specialist for schemas and migrations",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a database specialist. Use run_migration for schema changes.",
		Tools: []string{"run_migration"}, MaxTokens: &maxW,
	})
	uxDesigner, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "ux-designer", Description: "UX designer for accessibility and design",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a UX designer. Use review_design for accessibility checks.",
		Tools: []string{"review_design"}, MaxTokens: &maxW,
	})

	// Level 2: Team leads delegate to workers
	engineeringLead, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "engineering-lead", Description: "Engineering team lead coordinating technical work",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are the Engineering Team Lead. Coordinate frontend, backend, and database work.",
		AgentIDs: []uuid.UUID{frontendDev.ID, backendDev.ID, dbSpecialist.ID}, MaxTokens: &maxL,
	})
	designLead, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "design-lead", Description: "Design team lead coordinating UX work",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are the Design Team Lead. Coordinate design and UX work.",
		AgentIDs: []uuid.UUID{uxDesigner.ID}, MaxTokens: &maxL,
	})

	// Level 1: Project Manager delegates to leads
	pm, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "project-manager", Description: "Project manager coordinating all teams",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: `You are the Project Manager. Your direct reports:
- Engineering Lead: Manages frontend, backend, and database teams
- Design Lead: Manages UX and design team
Break tasks down and delegate appropriately.`,
		AgentIDs: []uuid.UUID{engineeringLead.ID, designLead.ID}, MaxTokens: &maxM,
	})

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Full project status - cascades through all 3 levels
	resp, _ := client.RunFastSync(ctx, sessionID, pm.ID,
		"Get a complete status update on the user auth feature. Check engineering on code quality and tests, and design on login page accessibility.", nil)
	if resp.Message != nil {
		for _, block := range resp.Message.Content {
			if block.Type == agentpg.ContentTypeText {
				fmt.Println(block.Text)
			}
		}
	}
}
```
