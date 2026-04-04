# Model Context Protocol (MCP) Integration

Claude Code supports the [Model Context Protocol](https://modelcontextprotocol.io/) in both directions — it can act as an **MCP client** (consuming tools from external servers) and as an **MCP server** (exposing its own tools to other clients).

---

## MCP Client (`src/services/mcpClient.ts`)

When Claude Code starts, it discovers and connects to configured MCP servers. Each server's tools become available to Claude alongside the built-in tools.

### Server Configuration

MCP servers can be configured in three scopes:

| Scope | File | Notes |
|-------|------|-------|
| `project` | `.claude/project.json` → `mcpServers` | Per-project, checked into the repo |
| `global` | `~/.claude/config.json` → `mcpServers` | User-level, applies to all projects |
| `mcprc` | `.mcprc` in working directory | Directory-local; requires explicit user approval |

### Server Types

**stdio** (local process):
```json
{
  "my-server": {
    "type": "stdio",
    "command": "node",
    "args": ["/path/to/server.js"],
    "env": { "API_KEY": "..." }
  }
}
```

**SSE** (HTTP Server-Sent Events):
```json
{
  "my-server": {
    "type": "sse",
    "url": "https://my-mcp-server.example.com/sse"
  }
}
```

### Tool Wrapping

Each tool exposed by an MCP server becomes an `MCPTool` instance in Claude Code. The tool:
- Carries the server name as a prefix in its `name` (e.g., `my-server__search`).
- Forwards `call()` invocations to the MCP server via the appropriate transport.
- Renders tool use/result UI in the terminal like any built-in tool.

### Prompt Wrapping

MCP servers can also expose **prompts** (pre-defined message templates). These become `PromptCommand` instances accessible as slash commands.

### Error Handling

MCP connection errors are logged via `logMCPError()` but do not crash Claude Code. If a server fails to connect, its tools are silently unavailable.

### `.mcprc` Security

`.mcprc` files are directory-local configs that could be committed by a third party. Before loading tools from a `.mcprc` server, Claude Code prompts the user for approval. Approved/rejected server IDs are stored in `projectConfig.approvedMcprcServers` / `rejectedMcprcServers`.

---

## MCP Server (`src/entrypoints/mcp.ts`)

Claude Code can itself act as an MCP server, exposing its built-in tools over the stdio MCP protocol. This is launched via:

```bash
claude --mcp-server
```

### Exposed Tools

The MCP server advertises these built-in tools:

- `AgentTool`
- `BashTool`
- `FileEditTool`
- `FileReadTool`
- `FileWriteTool`
- `GlobTool`
- `GrepTool`
- `lsTool`

### Protocol Handlers

| MCP Request | Handler |
|-------------|---------|
| `tools/list` | Returns all available tools with their JSON Schema input schemas |
| `tools/call` | Validates input, checks permissions, runs the tool, returns result |

### Permission Checking

Permission checks are enforced even when accessed via MCP. The server uses the same `hasPermissionsToUseTool` logic as the interactive REPL, but in non-interactive mode (no UI), denied permissions return an error result rather than prompting.

### State Tracking

`readFileTimestamps` is maintained by the MCP server to track when files were last read, enabling cache invalidation for tools that depend on file state.

---

## Data Flow: MCP Client Tool Call

```
Claude generates tool_use { name: "my-server__search", input: {...} }
  → MCPTool.call() invoked
  → StdioClientTransport / SSEClientTransport forwards to server
  → Server processes request, returns result
  → MCPTool yields ToolResult
  → Result appended as tool_result in next API call
```
