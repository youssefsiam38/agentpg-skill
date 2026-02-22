# AgentPG Skill

An [Agent Skill](https://agentskills.io/) that gives AI coding agents deep knowledge of the **AgentPG** framework — an event-driven, PostgreSQL-backed Go framework for building async AI agents with Claude integration.

## What Is This?

This is a skill package following the open [Agent Skills](https://agentskills.io/) specification. When installed, any compatible agent (Claude Code, Cursor, VS Code Copilot, Amp, Goose, and [many others](https://agentskills.io/#adoption)) gains framework-specific knowledge of AgentPG — APIs, types, patterns, and best practices — enabling it to write correct AgentPG code on the first try.

## What Is AgentPG?

[AgentPG](https://github.com/youssefsiam38/agentpg) is a Go framework for building AI agent systems where:

- **PostgreSQL is the backbone** — all state (sessions, runs, tool executions) is stored in Postgres, enabling distributed workers, crash recovery, and transactional guarantees.
- **Agents are async and event-driven** — runs are picked up by workers via `SELECT FOR UPDATE SKIP LOCKED`, with `LISTEN/NOTIFY` for real-time event propagation.
- **Claude is the LLM** — supports Batch API (50% cost discount) and Streaming API, with automatic tool orchestration.
- **Agent hierarchies** — agents can delegate to other agents (agent-as-tool pattern), enabling multi-level orchestration (PM → Tech Lead → Workers).
- **MCP integration** — connect external MCP servers (stdio or HTTP) and expose their tools to agents seamlessly.

## Skill Coverage

| Topic | File | What the Agent Learns |
|-------|------|-----------------------|
| **Core framework** | [SKILL.md](skills/agentpg/SKILL.md) | Client setup, lifecycle, Run/RunSync/RunFast APIs, RunOptions, response handling, cancel/regenerate, model selection, config defaults, error handling |
| **Tools** | [references/tools.md](skills/agentpg/references/tools.md) | `tool.Tool` interface, `ToolSchema`/`PropertyDef`, `FuncTool`, run context helpers (`GetVariable`, `GetRunContext`), error types (`ToolCancel`/`ToolDiscard`/`ToolSnooze`), retry config, database-aware tools |
| **MCP integration** | [references/mcp.md](skills/agentpg/references/mcp.md) | Stdio and HTTP transports, tool namespacing, `MCPServerConfig`, authentication options, error mapping, integration flow |
| **Agents** | [references/agents.md](skills/agentpg/references/agents.md) | Agent CRUD, `AgentDefinition` fields, agent-as-tool delegation, multi-level hierarchies, best practices |
| **Advanced features** | [references/advanced.md](skills/agentpg/references/advanced.md) | Distributed workers, tool-based routing, leader election, transaction-safe API, context compaction (auto/manual/strategies), admin UI, run rescue, LISTEN/NOTIFY events, monitoring queries, troubleshooting |
| **Examples** | [references/examples-*.md](skills/agentpg/references/) | Complete working examples for basic usage, tools, agent hierarchies, compaction, admin UI, retry/error handling, and cancel/regenerate flows |

## Installation

Place this skill where your agent can discover it. The exact location depends on your agent product:

### Claude Code

```bash
git clone https://github.com/youssefsiam38/agentpg-skill.git ~/.claude/skills/agentpg-skill
```

### Cursor / VS Code / Other Agents

Clone into your project's `.skills/` directory (or wherever your agent resolves skills from):

```bash
git clone https://github.com/youssefsiam38/agentpg-skill.git .skills/agentpg-skill
```

Refer to your agent's documentation for [skill installation details](https://agentskills.io/integrate-skills).

## Usage

Once installed, just ask your agent to work with AgentPG in natural language:

- *"Create a new AgentPG project with a weather tool and a researcher agent"*
- *"Add an MCP server for GitHub to my agent"*
- *"Set up distributed workers with tool-based routing"*
- *"Add context compaction to my long-running conversation agent"*
- *"Create a multi-level agent hierarchy with a PM, tech lead, and workers"*

The agent will reference the skill to produce correct, idiomatic AgentPG code.

## Repository Structure

```
skills/
  agentpg/
    SKILL.md                        # Main skill entry point
    references/
      tools.md                      # Tool interface reference
      mcp.md                        # MCP server integration
      agents.md                     # Agent management & delegation
      advanced.md                   # Distributed workers, transactions, compaction, UI
      examples-basic.md             # Getting started examples
      examples-tools.md             # Tool pattern examples
      examples-agents.md            # Agent hierarchy examples
      examples-compaction.md        # Context compaction examples
      examples-ui.md                # Admin UI examples
      examples-retry.md             # Error handling & retry examples
      examples-cancel.md            # Cancel & regenerate examples
```

## Contributing

Contributions are welcome! If you find gaps in the skill's coverage or want to add examples for new AgentPG features:

1. Fork the repository
2. Edit or add reference files under `skills/agentpg/references/`
3. Update `SKILL.md` if adding new sections or references
4. Submit a pull request

## License

MIT
