# Claude Code — Documentation

Generated documentation for the Claude Code v0.2.8 source code (extracted from the `@anthropic-ai/claude-code` npm package via source maps).

## Documents

| Doc | What it covers |
|-----|---------------|
| [architecture.md](./architecture.md) | Directory layout, component diagram, data flow, concurrency model |
| [how-claude-works.md](./how-claude-works.md) | Query loop, system prompt assembly, API calls, retry, tool execution |
| [memory.md](./memory.md) | All memory layers: session context, message history, memory files, config |
| [tools.md](./tools.md) | Every tool: purpose, permissions, input/output, special behavior |
| [permissions.md](./permissions.md) | Permission model, storage format, BashTool command matching, banned commands |
| [mcp.md](./mcp.md) | MCP client (consuming servers) and MCP server (exposing tools) |
| [configuration.md](./configuration.md) | Config files, environment variables, CLAUDE.md discovery |

## Quick Orientation

```
User types a prompt
  → REPL (src/screens/REPL.tsx)
    → query loop (src/query.ts) ←──────────────────┐
      → Claude API (src/services/claude.ts)         │
        → tool_use blocks                            │
          → tools (src/tools/*/)                     │
            → tool results ─────────────────────────┘
              (recurse until no more tool calls)
```

Claude's knowledge of the project comes from the system prompt assembled in `src/context.ts` at session start: git status, directory tree, `CLAUDE.md` files, and user-set context — all injected once and cached for the conversation.
