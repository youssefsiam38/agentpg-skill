# Examples: Custom Tools

## Table of Contents

- [01_struct_tool](#01_struct_tool) - Full tool.Tool interface with struct and internal state
- [02_func_tool](#02_func_tool) - Quick tool creation with tool.NewFuncTool
- [03_schema_validation](#03_schema_validation) - Advanced JSON Schema (Enum, Min/Max, Items, nested Properties)
- [04_parallel_execution](#04_parallel_execution) - Claude calling multiple tools in parallel

---

## 01_struct_tool

Full `tool.Tool` interface with struct, internal state, error handling.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"os"
	"os/signal"
	"syscall"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type WeatherTool struct {
	defaultUnit string
	locations   map[string]weatherData
}

type weatherData struct {
	Temperature float64
	Humidity    int
	Condition   string
}

func NewWeatherTool() *WeatherTool {
	return &WeatherTool{
		defaultUnit: "celsius",
		locations: map[string]weatherData{
			"new york":      {Temperature: 22.5, Humidity: 65, Condition: "partly cloudy"},
			"london":        {Temperature: 15.2, Humidity: 78, Condition: "rainy"},
			"tokyo":         {Temperature: 28.0, Humidity: 70, Condition: "sunny"},
			"sydney":        {Temperature: 19.8, Humidity: 55, Condition: "clear"},
			"paris":         {Temperature: 18.5, Humidity: 60, Condition: "cloudy"},
			"san francisco": {Temperature: 16.0, Humidity: 72, Condition: "foggy"},
		},
	}
}

func (w *WeatherTool) Name() string        { return "get_weather" }
func (w *WeatherTool) Description() string {
	return "Get current weather information for a specified city. Returns temperature, humidity, and conditions."
}

func (w *WeatherTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"city": {Type: "string", Description: "The city name (e.g., 'New York', 'London', 'Tokyo')"},
			"unit": {Type: "string", Description: "Temperature unit", Enum: []string{"celsius", "fahrenheit"}},
		},
		Required: []string{"city"},
	}
}

func (w *WeatherTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		City string `json:"city"`
		Unit string `json:"unit"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}
	if params.City == "" {
		return "", fmt.Errorf("city is required")
	}

	unit := params.Unit
	if unit == "" {
		unit = w.defaultUnit
	}

	cityLower := toLower(params.City)
	data, found := w.locations[cityLower]
	if !found {
		data = weatherData{
			Temperature: 15.0 + rand.Float64()*20.0,
			Humidity:    40 + rand.Intn(40),
			Condition:   []string{"sunny", "cloudy", "rainy", "partly cloudy"}[rand.Intn(4)],
		}
	}

	temp := data.Temperature
	if unit == "fahrenheit" {
		temp = temp*9/5 + 32
	}

	return fmt.Sprintf("Weather in %s:\n- Temperature: %.1f°%s\n- Humidity: %d%%\n- Conditions: %s",
		params.City, temp, string(unit[0]-32), data.Humidity, data.Condition), nil
}

func toLower(s string) string {
	result := make([]byte, len(s))
	for i := 0; i < len(s); i++ {
		c := s[i]
		if c >= 'A' && c <= 'Z' {
			c += 32
		}
		result[i] = c
	}
	return string(result)
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
	client, err := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: apiKey})
	if err != nil {
		log.Fatalf("Failed to create client: %v", err)
	}

	client.RegisterTool(NewWeatherTool())

	if err := client.Start(ctx); err != nil {
		log.Fatalf("Failed to start client: %v", err)
	}

	maxTokens := 1024
	agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "weather-assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful weather assistant. Use the get_weather tool to provide weather information when asked.",
		Tools:        []string{"get_weather"},
		MaxTokens:    &maxTokens,
	})
	if err != nil {
		log.Fatalf("Failed to create agent: %v", err)
	}
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, nil)

	response, _ := client.RunFastSync(ctx, sessionID, agent.ID, "What's the weather like in Tokyo?", nil)
	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```

---

## 02_func_tool

Quick tool creation with `tool.NewFuncTool` - no struct needed.

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

var timezoneOffsets = map[string]int{
	"UTC": 0, "EST": -5, "PST": -8, "CET": 1, "JST": 9, "AEST": 10,
	"GMT": 0, "IST": 5, "CST": -6, "MST": -7, "HST": -10,
}

func createTimeTool() tool.Tool {
	return tool.NewFuncTool(
		"get_time",
		"Get the current time in a specified timezone.",
		tool.ToolSchema{
			Type: "object",
			Properties: map[string]tool.PropertyDef{
				"timezone": {Type: "string", Description: "Timezone abbreviation (e.g., 'UTC', 'EST', 'PST', 'JST')"},
				"format":   {Type: "string", Description: "Output format", Enum: []string{"12h", "24h"}},
			},
			Required: []string{"timezone"},
		},
		func(ctx context.Context, input json.RawMessage) (string, error) {
			var params struct {
				Timezone string `json:"timezone"`
				Format   string `json:"format"`
			}
			if err := json.Unmarshal(input, &params); err != nil {
				return "", fmt.Errorf("invalid input: %w", err)
			}
			offset, found := timezoneOffsets[params.Timezone]
			if !found {
				return "", fmt.Errorf("unknown timezone: %s", params.Timezone)
			}
			now := time.Now().UTC().Add(time.Duration(offset) * time.Hour)
			var timeStr string
			if params.Format == "12h" {
				timeStr = now.Format("3:04:05 PM")
			} else {
				timeStr = now.Format("15:04:05")
			}
			dateStr := now.Format("Monday, January 2, 2006")
			return fmt.Sprintf("Current time in %s:\nTime: %s\nDate: %s\nUTC Offset: %+d hours",
				params.Timezone, timeStr, dateStr, offset), nil
		},
	)
}

func createDateDiffTool() tool.Tool {
	return tool.NewFuncTool(
		"calculate_date_diff",
		"Calculate the difference between two dates.",
		tool.ToolSchema{
			Type: "object",
			Properties: map[string]tool.PropertyDef{
				"start_date": {Type: "string", Description: "Start date in YYYY-MM-DD format"},
				"end_date":   {Type: "string", Description: "End date in YYYY-MM-DD format"},
				"unit":       {Type: "string", Description: "Unit for result", Enum: []string{"days", "weeks", "months"}},
			},
			Required: []string{"start_date", "end_date"},
		},
		func(ctx context.Context, input json.RawMessage) (string, error) {
			var params struct {
				StartDate string `json:"start_date"`
				EndDate   string `json:"end_date"`
				Unit      string `json:"unit"`
			}
			if err := json.Unmarshal(input, &params); err != nil {
				return "", fmt.Errorf("invalid input: %w", err)
			}
			start, err := time.Parse("2006-01-02", params.StartDate)
			if err != nil {
				return "", fmt.Errorf("invalid start_date format: %w", err)
			}
			end, err := time.Parse("2006-01-02", params.EndDate)
			if err != nil {
				return "", fmt.Errorf("invalid end_date format: %w", err)
			}
			days := int(end.Sub(start).Hours() / 24)
			unit := params.Unit
			if unit == "" {
				unit = "days"
			}
			var result string
			switch unit {
			case "days":
				result = fmt.Sprintf("%d days", days)
			case "weeks":
				result = fmt.Sprintf("%.1f weeks (%d days)", float64(days)/7.0, days)
			case "months":
				result = fmt.Sprintf("%.1f months (%d days)", float64(days)/30.44, days)
			}
			return fmt.Sprintf("Date difference from %s to %s:\n%s",
				params.StartDate, params.EndDate, result), nil
		},
	)
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY")})

	client.RegisterTool(createTimeTool())
	client.RegisterTool(createDateDiffTool())

	client.Start(ctx)

	maxTokens := 1024
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "time-assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful time assistant.",
		Tools:        []string{"get_time", "calculate_date_diff"},
		MaxTokens:    &maxTokens,
	})
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, nil)

	resp, _ := client.RunSync(ctx, sessionID, agent.ID, "What time is it in Tokyo right now?", nil)
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	resp, _ = client.RunSync(ctx, sessionID, agent.ID, "How many weeks are between 2024-01-01 and 2024-12-31?", nil)
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```

---

## 03_schema_validation

Advanced JSON Schema: Enum, Min/Max, MinLength/MaxLength, array Items, nested object Properties.

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
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type TaskTool struct {
	tasks []Task
}

type Task struct {
	ID       int
	Title    string
	Priority string
	Score    float64
	Tags     []string
	Assignee Assignee
}

type Assignee struct {
	Name  string
	Email string
}

func NewTaskTool() *TaskTool {
	return &TaskTool{tasks: make([]Task, 0)}
}

func (t *TaskTool) Name() string        { return "create_task" }
func (t *TaskTool) Description() string { return "Create a new task with priority, score, tags, and assignee." }

func (t *TaskTool) InputSchema() tool.ToolSchema {
	minScore := 0.0
	maxScore := 100.0
	minTitleLen := 3
	maxTitleLen := 100

	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"title": {
				Type: "string", Description: "Task title (3-100 characters)",
				MinLength: &minTitleLen, MaxLength: &maxTitleLen,
			},
			"description": {Type: "string", Description: "Detailed task description (optional)"},
			"priority": {
				Type: "string", Description: "Task priority level",
				Enum: []string{"low", "medium", "high", "critical"},
			},
			"score": {
				Type: "number", Description: "Task importance score from 0 to 100",
				Minimum: &minScore, Maximum: &maxScore,
			},
			"tags": {
				Type: "array", Description: "List of tags for categorization",
				Items: &tool.PropertyDef{Type: "string", Description: "A single tag"},
			},
			"assignee": {
				Type: "object", Description: "Person assigned to this task",
				Properties: map[string]tool.PropertyDef{
					"name":  {Type: "string", Description: "Assignee's full name"},
					"email": {Type: "string", Description: "Assignee's email address"},
				},
			},
		},
		Required: []string{"title", "priority"},
	}
}

func (t *TaskTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Title       string   `json:"title"`
		Description string   `json:"description"`
		Priority    string   `json:"priority"`
		Score       float64  `json:"score"`
		Tags        []string `json:"tags"`
		Assignee    struct {
			Name  string `json:"name"`
			Email string `json:"email"`
		} `json:"assignee"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", fmt.Errorf("invalid input: %w", err)
	}

	task := Task{
		ID: len(t.tasks) + 1, Title: params.Title,
		Priority: params.Priority, Score: params.Score, Tags: params.Tags,
	}
	if params.Assignee.Name != "" {
		task.Assignee = Assignee{Name: params.Assignee.Name, Email: params.Assignee.Email}
	}
	t.tasks = append(t.tasks, task)

	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("Task #%d created!\n- Title: %s\n- Priority: %s\n- Score: %.1f\n",
		task.ID, task.Title, task.Priority, task.Score))
	if len(task.Tags) > 0 {
		sb.WriteString(fmt.Sprintf("- Tags: %s\n", strings.Join(task.Tags, ", ")))
	}
	if task.Assignee.Name != "" {
		sb.WriteString(fmt.Sprintf("- Assignee: %s <%s>\n", task.Assignee.Name, task.Assignee.Email))
	}
	return sb.String(), nil
}

type ListTasksTool struct {
	taskTool *TaskTool
}

func NewListTasksTool(taskTool *TaskTool) *ListTasksTool {
	return &ListTasksTool{taskTool: taskTool}
}

func (l *ListTasksTool) Name() string        { return "list_tasks" }
func (l *ListTasksTool) Description() string { return "List all tasks, optionally filtered by priority" }
func (l *ListTasksTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"priority_filter": {Type: "string", Enum: []string{"low", "medium", "high", "critical"}},
		},
		Required: []string{},
	}
}
func (l *ListTasksTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		PriorityFilter string `json:"priority_filter"`
	}
	json.Unmarshal(input, &params)

	if len(l.taskTool.tasks) == 0 {
		return "No tasks found.", nil
	}
	var sb strings.Builder
	sb.WriteString("Tasks:\n")
	for _, task := range l.taskTool.tasks {
		if params.PriorityFilter != "" && task.Priority != params.PriorityFilter {
			continue
		}
		sb.WriteString(fmt.Sprintf("\n#%d: %s\n    Priority: %s | Score: %.1f\n",
			task.ID, task.Title, task.Priority, task.Score))
	}
	return sb.String(), nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY")})

	taskTool := NewTaskTool()
	client.RegisterTool(taskTool)
	client.RegisterTool(NewListTasksTool(taskTool))

	client.Start(ctx)
	_ = time.Now() // suppress unused import

	maxTokens := 1024
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "task-manager",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a task management assistant.",
		Tools:        []string{"create_task", "list_tasks"},
		MaxTokens:    &maxTokens,
	})
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, nil)

	// Create task with all fields including nested object and array
	resp, _ := client.RunFastSync(ctx, sessionID, agent.ID,
		`Create a high priority task called "Implement user authentication" with score 85, tags ["security", "backend"], and assign to John Smith (john@example.com).`, nil)
	for _, block := range resp.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}
}
```

---

## 04_parallel_execution

Claude calling multiple tools in parallel, tracking execution with DataFetchTool.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
)

type DataFetchTool struct {
	mu       sync.Mutex
	callLog  []string
	fetchLog []time.Time
}

func NewDataFetchTool() *DataFetchTool {
	return &DataFetchTool{callLog: make([]string, 0), fetchLog: make([]time.Time, 0)}
}

func (d *DataFetchTool) Name() string        { return "fetch_data" }
func (d *DataFetchTool) Description() string { return "Fetch data from a specified source" }
func (d *DataFetchTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"source": {Type: "string", Enum: []string{"database", "api", "cache", "file"}},
			"query":  {Type: "string", Description: "Query or identifier for the data"},
		},
		Required: []string{"source", "query"},
	}
}

func (d *DataFetchTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var params struct {
		Source string `json:"source"`
		Query  string `json:"query"`
	}
	if err := json.Unmarshal(input, &params); err != nil {
		return "", err
	}

	d.mu.Lock()
	d.callLog = append(d.callLog, fmt.Sprintf("%s:%s", params.Source, params.Query))
	d.fetchLog = append(d.fetchLog, time.Now())
	d.mu.Unlock()

	// Simulate different fetch times
	switch params.Source {
	case "cache":
		time.Sleep(10 * time.Millisecond)
	case "database":
		time.Sleep(50 * time.Millisecond)
	case "api":
		time.Sleep(100 * time.Millisecond)
	}

	data := map[string]string{
		"database": fmt.Sprintf("DB record for '%s': {id: 123, status: 'active'}", params.Query),
		"api":      fmt.Sprintf("API response for '%s': {data: 'fetched'}", params.Query),
		"cache":    fmt.Sprintf("Cached value for '%s': 'cached_data_v1'", params.Query),
	}
	return data[params.Source], nil
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{APIKey: os.Getenv("ANTHROPIC_API_KEY")})

	dataFetcher := NewDataFetchTool()
	client.RegisterTool(dataFetcher)

	client.Start(ctx)

	maxTokens := 2048
	agent, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "data-processor",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a data processing assistant. When asked to fetch from multiple sources, call all the fetch_data tools in parallel.",
		MaxTokens:    &maxTokens,
		Tools:        []string{"fetch_data"},
	})
	defer client.Stop(context.Background())

	sessionID, _ := client.NewSession(ctx, nil, nil)

	start := time.Now()
	response, _ := client.RunFastSync(ctx, sessionID, agent.ID,
		"Fetch the user profile from the database, the cache, and the API. The query for all should be 'user-123'. Call all three fetch_data tools in your response.", nil)
	totalDuration := time.Since(start)

	for _, block := range response.Message.Content {
		if block.Type == agentpg.ContentTypeText {
			fmt.Println(block.Text)
		}
	}

	fmt.Printf("\nTotal time: %v\n", totalDuration)
	fmt.Printf("Iterations: %d, Tool iterations: %d\n", response.IterationCount, response.ToolIterations)

	// Show parallel execution timing
	callLog := dataFetcher.callLog
	fetchTimes := dataFetcher.fetchLog
	if len(callLog) > 0 {
		firstCall := fetchTimes[0]
		for i, call := range callLog {
			offset := fetchTimes[i].Sub(firstCall)
			fmt.Printf("%d. %s (started at +%v)\n", i+1, call, offset)
		}
	}
}
```
