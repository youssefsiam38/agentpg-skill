# AgentPG MCP Server Tools Reference

## Table of Contents

- [Installation](#installation)
- [Stdio Transport](#stdio-transport)
- [HTTP Transport](#http-transport)
- [Tool Namespacing](#tool-namespacing)
- [Configuration Types](#configuration-types)
- [Error Mapping](#error-mapping)
- [Integration Flow](#integration-flow)

## Installation

```bash
go get github.com/youssefsiam38/agentpg/mcp
```

```go
import agentmcp "github.com/youssefsiam38/agentpg/mcp"
```

The `mcp` sub-module discovers tools from external [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers and bridges them into AgentPG's `tool.Tool` interface.

## Stdio Transport

```go
mcpServer, err := agentmcp.RegisterServer(ctx, client, &agentmcp.MCPServerConfig{
    Name: "github",
    Stdio: &agentmcp.StdioTransportConfig{
        Command: "npx",
        Args:    []string{"-y", "@modelcontextprotocol/server-github"},
        Env:     []string{"GITHUB_PERSONAL_ACCESS_TOKEN=" + token},
    },
    ToolFilter: func(name string) bool {
        return name == "search_repositories" || name == "get_file_contents"
    },
})
if err != nil { log.Fatal(err) }
defer mcpServer.Close()
```

## HTTP Transport

```go
mcpServer, err := agentmcp.RegisterServer(ctx, client, &agentmcp.MCPServerConfig{
    Name: "api",
    HTTP: &agentmcp.HTTPTransportConfig{
        URL: "https://mcp-server.internal/mcp",
        HeaderFunc: func() (map[string]string, error) {
            token, err := oauthProvider.FreshToken()
            return map[string]string{"Authorization": "Bearer " + token}, err
        },
    },
})
defer mcpServer.Close()
```

## Tool Namespacing

Tools are prefixed with `{serverName}__{toolName}` by default:
- `github` server, `search_code` tool -> `github__search_code`
- `fs` server, `read_file` tool -> `fs__read_file`

Set `DisableToolPrefix: true` to use original names (risk of collision).

## Configuration Types

```go
type MCPServerConfig struct {
    Name              string                     // Required. Namespace prefix
    Stdio             *StdioTransportConfig      // Mutually exclusive with HTTP
    HTTP              *HTTPTransportConfig       // Mutually exclusive with Stdio
    DisableToolPrefix bool                       // Default: false
    ToolFilter        func(toolName string) bool // Optional: filter tools
    Reconnect         *ReconnectConfig           // Optional: auto-reconnect
}

type StdioTransportConfig struct {
    Command string   // MCP server executable
    Args    []string
    Env     []string // Appended to os.Environ()
    Dir     string   // Working directory
}

type HTTPTransportConfig struct {
    URL        string
    HTTPClient *http.Client
    Headers    map[string]string                   // Static headers
    HeaderFunc func() (map[string]string, error)   // Dynamic headers (overrides static)
}

type ReconnectConfig struct {
    MaxRetries   int           // 0 = unlimited
    InitialDelay time.Duration // Default: 1s
    MaxDelay     time.Duration // Default: 30s
}
```

### Authentication Options

| Method | Use Case |
|--------|----------|
| `StdioTransportConfig.Env` | API keys/tokens as env vars to subprocess |
| `HTTPTransportConfig.Headers` | Static auth headers |
| `HTTPTransportConfig.HeaderFunc` | Dynamic/rotating tokens |
| `HTTPTransportConfig.HTTPClient` | Full control: mTLS, custom RoundTripper |

## Error Mapping

MCP errors are automatically mapped to AgentPG tool error types:

| MCP Error | AgentPG Mapping | Behavior |
|-----------|-----------------|----------|
| Connection refused/timeout/EOF | `tool.ToolSnooze(10s)` | Retry after delay |
| Rate limited (429) | `tool.ToolSnooze(30s)` | Retry after longer delay |
| Auth errors (401, 403) | `tool.ToolCancel` | No retry |
| Invalid params | `tool.ToolDiscard` | No retry |
| MCP `isError: true` result | Returned as tool result text | Claude sees error and adjusts |

## Integration Flow

```go
// 1. Create client
client, _ := agentpg.NewClient(drv, config)

// 2. Register local tools (optional)
client.RegisterTool(&LocalTool{})

// 3. Register MCP server (BEFORE Start)
mcpServer, _ := agentmcp.RegisterServer(ctx, client, &agentmcp.MCPServerConfig{
    Name: "github",
    Stdio: &agentmcp.StdioTransportConfig{
        Command: "npx",
        Args:    []string{"-y", "@modelcontextprotocol/server-github"},
        Env:     []string{"GITHUB_PERSONAL_ACCESS_TOKEN=" + token},
    },
})
defer mcpServer.Close()

// 4. Start client (syncs all tools to database)
client.Start(ctx)
defer client.Stop(context.Background())

// 5. Collect MCP tool names for agent
var toolNames []string
for _, t := range mcpServer.Tools() {
    toolNames = append(toolNames, t.Name())
}

// 6. Create agent with MCP tools
agent, _ := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
    Name:  "github-assistant",
    Model: "claude-sonnet-4-5-20250929",
    Tools: toolNames, // e.g. ["github__search_repositories", "github__get_file_contents"]
})

// 7. Run — agent uses MCP tools transparently
sessionID, _ := client.NewSession(ctx, nil, nil)
response, _ := client.RunFastSync(ctx, sessionID, agent.ID, "Search for Go repos about AI agents", nil)
```

**Important**: `RegisterServer()` must be called **before** `client.Start()`.
