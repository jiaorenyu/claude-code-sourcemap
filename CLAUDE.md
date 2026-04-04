# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is an **extracted source code archive** of Claude Code v0.2.8 (`@anthropic-ai/claude-code`), reconstructed from the published npm package using source maps. It is a **read-only reference** — there is no build system, no `package.json` at the root, and no test runner. The compiled output lives in `cli.mjs` (23MB bundle). The fork under active development is at https://github.com/dnakov/anon-kode.

`vendor/sdk/` contains a vendored copy of `@anthropic-ai/sdk` v0.36.3.

## Architecture Overview

Claude Code is a terminal-based AI coding agent built with React/[Ink](https://github.com/vadimdemedes/ink) for terminal UI rendering and the Anthropic SDK for Claude API calls.

### Main Data Flow

1. **Entry**: `src/entrypoints/cli.tsx` — parses CLI args, initializes config, mounts the REPL
2. **REPL**: `src/screens/REPL.tsx` — React component managing the interactive session and message history
3. **Query loop**: `src/query.ts` — orchestrates calls to Claude, dispatches tool use with up to 10 concurrent tool invocations (`MAX_TOOL_USE_CONCURRENCY`), and recurses until no further tool calls are needed
4. **Claude API**: `src/services/claude.ts` — wraps the Anthropic SDK; handles model selection (Sonnet for main, Haiku for lightweight ops), prompt caching, cost tracking, and exponential backoff retry (max 10 retries, up to 32s delay)
5. **Tools**: `src/tools/*/` — each tool exports an object conforming to the `Tool` interface in `src/Tool.ts`
6. **Rendering**: results stream back through React components in `src/components/`

### Key Modules

| Path | Role |
|------|------|
| `src/query.ts` | Core query/tool-use orchestration loop |
| `src/services/claude.ts` | Anthropic SDK wrapper, retry, cost tracking |
| `src/context.ts` | Assembles system prompt context (git status, CLAUDE.md files, directory tree) |
| `src/permissions.ts` | Per-tool permission gating; filesystem permission checks |
| `src/commands.ts` | Slash command registry (`/help`, `/bug`, `/config`, etc.) |
| `src/tools.ts` | Tool registry and initialization |
| `src/history.ts` | Conversation history persistence |
| `src/cost-tracker.ts` | Token cost accumulation and budget warnings |
| `src/services/oauth.ts` | Anthropic Console OAuth flow |
| `src/services/mcpClient.ts` | MCP (Model Context Protocol) client for custom tools |
| `src/services/statsig.ts` | Feature gates and telemetry |
| `src/entrypoints/mcp.ts` | MCP server entry point |

### Tool Interface

Every tool in `src/tools/*/` implements the `Tool` interface (`src/Tool.ts`):
- `name` — identifier sent to Claude API
- `inputSchema` — Zod schema for input validation
- `call()` — async generator yielding tool results
- `isReadOnly()` / `isEnabled()` / `needsPermissions()` — runtime capability checks
- `renderToolUseMessage()` / `renderToolResultMessage()` — Ink/React renderers for terminal UI

### Available Tools (18)

`AgentTool`, `ArchitectTool`, `BashTool`, `FileEditTool`, `FileReadTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `lsTool`, `MCPTool`, `MemoryReadTool`, `MemoryWriteTool`, `NotebookEditTool`, `NotebookReadTool`, `StickerRequestTool`, `ThinkTool` — plus MCP-proxied tools.

`MemoryReadTool` / `MemoryWriteTool` and `StickerRequestTool` are Anthropic-internal tools gated behind `USER_TYPE === 'ant'`.

### State & Configuration

- User config and session logs: `~/.claude/`
- Current working directory tracked in `src/utils/state.ts` via `getCwd()`
- Feature flags via Statsig (`src/services/statsig.ts`); beta features (Think Tool, Binary Feedback) gated there
- AWS Bedrock / Google Vertex support selectable via environment flags

### Message Types

`src/query.ts` exports three message shapes used throughout the UI:
- `UserMessage` — user input, optionally carrying a `FullToolUseResult`
- `AssistantMessage` — Claude response with cost and duration metadata
- `ProgressMessage` — intermediate state during concurrent tool execution

## Detailed Documentation

See [`docs/`](./docs/README.md) for full documentation:

- [`docs/architecture.md`](./docs/architecture.md) — component diagram, data flow, concurrency
- [`docs/how-claude-works.md`](./docs/how-claude-works.md) — query loop, system prompt, API calls, retry
- [`docs/memory.md`](./docs/memory.md) — all memory layers and what survives a restart
- [`docs/tools.md`](./docs/tools.md) — every tool's behavior, permissions, and I/O
- [`docs/permissions.md`](./docs/permissions.md) — permission model, storage, BashTool command matching
- [`docs/mcp.md`](./docs/mcp.md) — MCP client and server integration
- [`docs/configuration.md`](./docs/configuration.md) — config files, env vars, CLAUDE.md discovery
