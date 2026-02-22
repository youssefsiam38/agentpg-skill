# Examples: Admin UI

## Table of Contents

- [admin_ui](#admin_ui) - Basic UI embedding
- [admin_ui_auth](#admin_ui_auth) - Authentication middleware
- [admin_ui_full](#admin_ui_full) - Full-featured with multiple agents, tools, and read-only monitoring

---

## admin_ui

Basic admin UI embedding with `ui.UIHandler`.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/ui"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() { <-sigCh; cancel() }()

	dbURL := os.Getenv("DATABASE_URL")
	if dbURL == "" {
		dbURL = "postgres://agentpg:agentpg@localhost:5432/agentpg?sslmode=disable"
	}

	pool, _ := pgxpool.New(ctx, dbURL)
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"),
		Name:   "admin-ui-example",
	})

	client.Start(ctx)
	defer client.Stop(context.Background())

	client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name:         "assistant",
		Description:  "A helpful AI assistant",
		Model:        "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful AI assistant. Be concise and friendly.",
	})

	mux := http.NewServeMux()

	// Mount the admin UI at /ui/
	uiHandler := ui.UIHandler(drv.Store(), client, &ui.Config{
		PageSize:        25,
		RefreshInterval: 5 * time.Second,
	})
	mux.Handle("/ui/", http.StripPrefix("/ui", uiHandler))

	// App routes
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/" {
			http.NotFound(w, r)
			return
		}
		fmt.Fprintf(w, `<html><body>
<h1>AgentPG Admin UI Example</h1>
<ul>
  <li><a href="/ui/">Dashboard</a></li>
  <li><a href="/ui/sessions">Sessions</a></li>
  <li><a href="/ui/runs">Runs</a></li>
  <li><a href="/ui/agents">Agents</a></li>
  <li><a href="/ui/chat">Chat</a></li>
</ul>
</body></html>`)
	})

	server := &http.Server{Addr: ":8080", Handler: mux}
	go func() {
		log.Println("Server on http://localhost:8080, Admin UI at http://localhost:8080/ui/")
		server.ListenAndServe()
	}()

	<-ctx.Done()
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	server.Shutdown(shutdownCtx)
}
```

---

## admin_ui_auth

Authentication middleware with bcrypt, session cookies, login/logout flow.

Default credentials: `admin/admin123`, `demo/demo123`.

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
	"github.com/youssefsiam38/agentpg/ui"
	"golang.org/x/crypto/bcrypt"
)

type Session struct {
	Username  string
	CreatedAt time.Time
}

var (
	users = map[string]string{
		"admin": "$2a$10$tgo2gAgGgTbTzydKAQQFeeuVZ.UOHwiNuB56S2FkYV44CIRv7IXZu", // admin123
		"demo":  "$2a$10$k9.gPoO62SZCJ1S8mf2DuuFkku61lqni77lBYtO4iemvcGkdRhKc6", // demo123
	}
	sessions sync.Map
)

func generateSessionToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return hex.EncodeToString(b)
}

func createSession(username string) string {
	token := generateSessionToken()
	sessions.Store(token, &Session{Username: username, CreatedAt: time.Now()})
	return token
}

func getSession(token string) *Session {
	if v, ok := sessions.Load(token); ok {
		return v.(*Session)
	}
	return nil
}

// authMiddleware wraps handlers requiring authentication
func authMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		cookie, err := r.Cookie("agentpg_session")
		if err != nil || getSession(cookie.Value) == nil {
			http.Error(w, `{"error":"unauthorized"}`, http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func handleLogin(w http.ResponseWriter, r *http.Request) {
	if r.Method == http.MethodGet {
		// Render login form HTML
		w.Header().Set("Content-Type", "text/html")
		fmt.Fprint(w, `<form method="POST" action="/login">
			<input name="username" required><input name="password" type="password" required>
			<button type="submit">Sign in</button></form>`)
		return
	}

	r.ParseForm()
	username := r.FormValue("username")
	password := r.FormValue("password")
	nextURL := r.FormValue("next")

	hash, ok := users[username]
	if !ok || bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) != nil {
		redirectURL := "/login?error=invalid"
		if nextURL != "" {
			redirectURL += "&next=" + url.QueryEscape(nextURL)
		}
		http.Redirect(w, r, redirectURL, http.StatusFound)
		return
	}

	token := createSession(username)
	http.SetCookie(w, &http.Cookie{
		Name: "agentpg_session", Value: token, Path: "/",
		HttpOnly: true, MaxAge: 86400, SameSite: http.SameSiteLaxMode,
	})

	if nextURL == "" {
		nextURL = "/ui/dashboard"
	}
	http.Redirect(w, r, nextURL, http.StatusFound)
}

func handleLogout(w http.ResponseWriter, r *http.Request) {
	if cookie, err := r.Cookie("agentpg_session"); err == nil {
		sessions.Delete(cookie.Value)
	}
	http.SetCookie(w, &http.Cookie{Name: "agentpg_session", Value: "", Path: "/", MaxAge: -1})
	http.Redirect(w, r, "/login", http.StatusFound)
}

type CalculatorTool struct{}
func (t *CalculatorTool) Name() string        { return "calculator" }
func (t *CalculatorTool) Description() string { return "Perform simple arithmetic" }
func (t *CalculatorTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{
		Type: "object",
		Properties: map[string]tool.PropertyDef{
			"a": {Type: "number"}, "b": {Type: "number"},
			"op": {Type: "string", Enum: []string{"add", "subtract", "multiply", "divide"}},
		},
		Required: []string{"a", "b", "op"},
	}
}
func (t *CalculatorTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var p struct{ A, B float64; Op string }
	json.Unmarshal(input, &p)
	var r float64
	switch p.Op {
	case "add": r = p.A + p.B
	case "subtract": r = p.A - p.B
	case "multiply": r = p.A * p.B
	case "divide": r = p.A / p.B
	}
	return fmt.Sprintf("%g", r), nil
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() { <-sigCh; cancel() }()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"), Name: "admin-ui-auth-example",
	})

	client.RegisterTool(&CalculatorTool{})
	client.Start(ctx)

	client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "assistant", Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful AI assistant with calculator capabilities.",
		Tools: []string{"calculator"},
	})
	defer client.Stop(context.Background())

	mux := http.NewServeMux()

	// Public routes
	mux.HandleFunc("/login", handleLogin)
	mux.HandleFunc("/logout", handleLogout)

	// Protected UI - wrapped with authMiddleware
	uiConfig := &ui.Config{BasePath: "/ui", PageSize: 25, RefreshInterval: 5 * time.Second}
	mux.Handle("/ui/", http.StripPrefix("/ui", authMiddleware(ui.UIHandler(drv.Store(), client, uiConfig))))

	server := &http.Server{Addr: ":8080", Handler: mux}
	go server.ListenAndServe()

	<-ctx.Done()
	shutdownCtx, c := context.WithTimeout(context.Background(), 10*time.Second)
	defer c()
	server.Shutdown(shutdownCtx)
}
```

---

## admin_ui_full

Full-featured: multiple agents, custom tools, agent-as-tool, chat, read-only monitoring endpoint.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"log/slog"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/youssefsiam38/agentpg"
	"github.com/youssefsiam38/agentpg/driver/pgxv5"
	"github.com/youssefsiam38/agentpg/tool"
	"github.com/youssefsiam38/agentpg/ui"
)

// Tools: calculator, web_search, get_weather, database_query (implementations omitted for brevity)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() { <-sigCh; cancel() }()

	pool, _ := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
	defer pool.Close()

	drv := pgxv5.New(pool)
	client, _ := agentpg.NewClient(drv, &agentpg.ClientConfig{
		APIKey: os.Getenv("ANTHROPIC_API_KEY"), Name: "admin-ui-full-example",
		MaxConcurrentRuns: 5,
	})

	// Register tools: calculator, web_search, get_weather, database_query
	client.RegisterTool(&CalculatorTool{})
	client.RegisterTool(&WebSearchTool{})
	client.RegisterTool(&WeatherTool{})
	client.RegisterTool(&DatabaseQueryTool{})

	client.Start(ctx)
	defer client.Stop(context.Background())

	// Create agents
	assistant, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "assistant", Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "You are a helpful AI assistant.",
		Tools: []string{"calculator"},
	})

	researcher, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "researcher", Description: "Research specialist with web search",
		Model: "claude-sonnet-4-5-20250929",
		Tools: []string{"web_search"},
	})

	analyst, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "analyst", Description: "Data analyst with calculator",
		Model: "claude-sonnet-4-5-20250929",
		Tools: []string{"calculator", "database_query"},
	})

	weatherAssistant, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "weather_assistant", Description: "Weather information specialist",
		Model: "claude-3-5-haiku-20241022", // Fast model for simple tasks
		Tools: []string{"get_weather"},
	})

	// Coordinator delegates to specialists via AgentIDs
	client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
		Name: "coordinator", Description: "Orchestrates other agents",
		Model: "claude-sonnet-4-5-20250929",
		SystemPrompt: "Delegate research to researcher, data analysis to analyst, weather to weather_assistant.",
		AgentIDs: []uuid.UUID{researcher.ID, analyst.ID, weatherAssistant.ID},
	})

	mux := http.NewServeMux()

	// Full UI with chat
	fullUIConfig := &ui.Config{
		BasePath:           "/ui",
		PageSize:           25,
		RefreshInterval:    5 * time.Second,
		MetadataFilterKeys: []string{"tenant_id", "user_id"},
		Logger:             slog.New(slog.NewTextHandler(os.Stdout, nil)),
	}
	mux.Handle("/ui/", http.StripPrefix("/ui", ui.UIHandler(drv.Store(), client, fullUIConfig)))

	// Read-only monitoring UI (no chat, no write operations)
	monitorConfig := &ui.Config{
		BasePath:        "/monitor",
		ReadOnly:        true,
		PageSize:        50,
		RefreshInterval: 10 * time.Second,
	}
	mux.Handle("/monitor/", http.StripPrefix("/monitor", ui.UIHandler(drv.Store(), nil, monitorConfig)))

	// Demo endpoint
	mux.HandleFunc("POST /demo/create-session", func(w http.ResponseWriter, r *http.Request) {
		sessionID, _ := client.NewSession(r.Context(), nil, map[string]any{
			"tenant_id": fmt.Sprintf("demo-%d", time.Now().Unix()%1000),
			"demo":      true,
		})
		runID, _ := client.RunFast(r.Context(), sessionID, assistant.ID,
			"Hello! Please introduce yourself.", nil)
		http.Redirect(w, r, fmt.Sprintf("/ui/sessions/%s?run=%s", sessionID, runID), http.StatusSeeOther)
	})

	server := &http.Server{Addr: ":8090", Handler: mux}
	go func() {
		log.Println("Server on http://localhost:8090")
		log.Println("  /ui/      - Full admin UI with chat")
		log.Println("  /monitor/ - Read-only monitoring")
		server.ListenAndServe()
	}()

	<-ctx.Done()
	shutdownCtx, c := context.WithTimeout(context.Background(), 10*time.Second)
	defer c()
	server.Shutdown(shutdownCtx)
}

// Tool implementations (abbreviated)
type CalculatorTool struct{}
func (t *CalculatorTool) Name() string { return "calculator" }
func (t *CalculatorTool) Description() string { return "Perform mathematical calculations" }
func (t *CalculatorTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{Type: "object", Properties: map[string]tool.PropertyDef{
		"expression": {Type: "string", Description: "Math expression to evaluate"},
	}, Required: []string{"expression"}}
}
func (t *CalculatorTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var p struct{ Expression string `json:"expression"` }
	json.Unmarshal(input, &p)
	return fmt.Sprintf("Result: %s (evaluated)", p.Expression), nil
}

type WebSearchTool struct{}
func (t *WebSearchTool) Name() string { return "web_search" }
func (t *WebSearchTool) Description() string { return "Search the web" }
func (t *WebSearchTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{Type: "object", Properties: map[string]tool.PropertyDef{
		"query": {Type: "string"},
	}, Required: []string{"query"}}
}
func (t *WebSearchTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var p struct{ Query string `json:"query"` }
	json.Unmarshal(input, &p)
	return fmt.Sprintf("Search results for '%s': 1. Wikipedia 2. News 3. Research", p.Query), nil
}

type WeatherTool struct{}
func (t *WeatherTool) Name() string { return "get_weather" }
func (t *WeatherTool) Description() string { return "Get current weather" }
func (t *WeatherTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{Type: "object", Properties: map[string]tool.PropertyDef{
		"location": {Type: "string"},
	}, Required: []string{"location"}}
}
func (t *WeatherTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	var p struct{ Location string `json:"location"` }
	json.Unmarshal(input, &p)
	return fmt.Sprintf("Weather for %s: %dF, %s", p.Location,
		68+rand.Intn(12), []string{"Sunny", "Cloudy", "Clear"}[rand.Intn(3)]), nil
}

type DatabaseQueryTool struct{}
func (t *DatabaseQueryTool) Name() string { return "database_query" }
func (t *DatabaseQueryTool) Description() string { return "Query a database" }
func (t *DatabaseQueryTool) InputSchema() tool.ToolSchema {
	return tool.ToolSchema{Type: "object", Properties: map[string]tool.PropertyDef{
		"query": {Type: "string"},
	}, Required: []string{"query"}}
}
func (t *DatabaseQueryTool) Execute(ctx context.Context, input json.RawMessage) (string, error) {
	return "Results: 3 rows returned", nil
}
```
