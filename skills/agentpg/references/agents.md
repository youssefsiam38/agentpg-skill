# AgentPG Agents Reference

## Table of Contents

- [Agent CRUD](#agent-crud)
- [AgentDefinition Fields](#agentdefinition-fields)
- [Agent-as-Tool Pattern](#agent-as-tool-pattern)
- [Multi-Level Hierarchies](#multi-level-hierarchies)
- [Best Practices](#best-practices)

## Agent CRUD

Agents are database entities with UUID primary keys. Manage after `client.Start()`.

```go
// Create
agent, err := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:         "assistant",
    Model:        "claude-sonnet-4-5-20250929",
    SystemPrompt: "You are a helpful assistant.",
    Tools:        []string{"calculator"},
    Metadata:     map[string]any{"tenant_id": "t1"},
})

// Idempotent create (safe for every startup)
agent, err := client.GetOrCreateAgent(ctx, &agentpg.AgentDefinition{
    Name:  "assistant",
    Model: "claude-sonnet-4-5-20250929",
})

// Query
agent, err := client.GetAgentByID(ctx, agentID)
agent, err := client.GetAgentByName(ctx, "assistant", nil)                              // No metadata filter
agent, err := client.GetAgentByName(ctx, "assistant", map[string]any{"tenant_id": "t1"}) // With metadata
agents, total, err := client.ListAgents(ctx, nil, 100, 0)                               // All, limit 100
agents, total, err := client.ListAgents(ctx, map[string]any{"tenant_id": "t1"}, 100, 0) // Filtered

// Update
agent.SystemPrompt = "Updated prompt."
err := client.UpdateAgent(ctx, agent)

// Delete
err := client.DeleteAgent(ctx, agentID)
```

## AgentDefinition Fields

```go
type AgentDefinition struct {
    ID           uuid.UUID          // Auto-generated
    Name         string             // Required, unique display name
    Description  string             // Required when used as tool (shown to parent agent)
    Model        string             // Required: Claude model ID
    SystemPrompt string
    Tools        []string           // Tool names this agent can use
    AgentIDs     []uuid.UUID        // Agent UUIDs for delegation (agent-as-tool)
    MaxTokens    *int               // Max response tokens
    Temperature  *float64
    TopK         *int
    TopP         *float64
    Metadata     map[string]any     // App-specific (tenant_id, tags, etc.)
    Config       map[string]any     // Additional settings
}
```

## Agent-as-Tool Pattern

One agent delegates work to another. The child agent appears as a callable tool to the parent.

```go
// 1. Create child agent (specialist)
researcher, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:         "researcher",
    Description:  "Research specialist for gathering and analyzing information",  // Shown to parent
    Model:        "claude-sonnet-4-5-20250929",
    SystemPrompt: "You are a research specialist. Provide thorough analysis.",
})

// 2. Create parent with delegation via AgentIDs
manager, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:         "manager",
    Model:        "claude-sonnet-4-5-20250929",
    SystemPrompt: "Delegate research tasks to your researcher tool.",
    AgentIDs:     []uuid.UUID{researcher.ID},  // researcher becomes a tool
})

// 3. Run parent - it can invoke researcher as a tool
response, _ := client.RunSync(ctx, sessionID, manager.ID, "Research quantum computing", nil)
```

When manager calls the researcher "tool", AgentPG creates a child run for the researcher agent. The child run gets its own session (child of the parent's session).

## Multi-Level Hierarchies

```go
// Level 3: Workers with specialized tools
frontendDev, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:        "frontend-dev",
    Description: "Frontend developer - implements UI components",
    Model:       "claude-sonnet-4-5-20250929",
    Tools:       []string{"lint", "format_code"},
})

backendDev, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:        "backend-dev",
    Description: "Backend developer - implements APIs and services",
    Model:       "claude-sonnet-4-5-20250929",
    Tools:       []string{"run_tests", "query_db"},
})

// Level 2: Team lead delegates to workers
techLead, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:        "tech-lead",
    Description: "Technical lead - coordinates development work",
    Model:       "claude-sonnet-4-5-20250929",
    AgentIDs:    []uuid.UUID{frontendDev.ID, backendDev.ID},
})

// Level 1: PM delegates to tech lead
pm, _ := client.CreateAgent(ctx, &agentpg.AgentDefinition{
    Name:     "project-manager",
    Model:    "claude-opus-4-5-20251101",  // More capable model for orchestration
    AgentIDs: []uuid.UUID{techLead.ID},
})
```

An agent can have BOTH `Tools` (regular tools) and `AgentIDs` (agent delegation) simultaneously.

## Best Practices

- **Single responsibility**: Each agent should have one clear role
- **Clear descriptions**: Required for child agents - the parent sees this to decide when to delegate
- **Model selection**: Use cheaper models (`haiku`) for simple specialist tasks, capable models (`opus`) for orchestrators
- **Limit depth**: Each delegation level = additional API call. Keep hierarchies shallow (2-3 levels max)
- **Idempotent creation**: Use `GetOrCreateAgent()` in startup code for safety
- **Variable propagation**: Run variables automatically flow from parent to child runs. Instruction overrides (`OverrideInstructions`, `AppendInstructions`) do NOT propagate — each child agent uses its own system prompt.
