# Architecture Overview

Claude Code is a terminal-based AI coding agent. Its source is a TypeScript/React application bundled into a single `cli.mjs` file. This document describes the high-level architecture and how the major subsystems fit together.

---

## Directory Structure

```
src/
├── entrypoints/        # CLI and MCP server entry points
├── tools/              # Tool implementations (one dir per tool)
├── commands/           # Slash command handlers
├── services/           # External service integrations (Claude API, OAuth, MCP, Statsig)
├── screens/            # Full-screen React/Ink UI components (REPL, LogList, Doctor)
├── components/         # Reusable UI components and message renderers
├── hooks/              # React hooks for REPL state management
├── utils/              # Pure utilities (config, logging, permissions, messages, etc.)
└── constants/          # Static data: prompts, OAuth endpoints, pricing, beta gate names
```

---

## Component Diagram

```
┌─────────────────────────────────────────────────────┐
│                    cli.tsx (entry)                   │
│  arg parse → config init → OAuth check → REPL mount │
└───────────────────────┬─────────────────────────────┘
                        │ renders
                        ▼
┌─────────────────────────────────────────────────────┐
│                  REPL.tsx (screen)                   │
│  message list · input box · status bar · cost        │
└──────────┬────────────────────────────┬─────────────┘
           │ user submits               │ renders messages
           ▼                            ▼
┌────────────────────┐      ┌───────────────────────────┐
│     query.ts       │      │  components/messages/     │
│  main query loop   │      │  AssistantMessage         │
│  tool orchestration│      │  ToolUseMessage           │
└──────┬─────────────┘      │  ToolResultMessage        │
       │                    └───────────────────────────┘
       │ API call
       ▼
┌────────────────────┐
│  services/claude.ts│
│  Anthropic SDK     │
│  retry / caching   │
└──────┬─────────────┘
       │ tool_use blocks
       ▼
┌────────────────────────────────────────────────────┐
│                   tools/*/                          │
│  BashTool · FileEditTool · GlobTool · GrepTool …   │
│  Runs concurrently (read-only) or serially (write) │
└────────────────────────────────────────────────────┘
```

---

## Subsystem Responsibilities

| Subsystem | Files | Responsibility |
|-----------|-------|----------------|
| Entry | `entrypoints/cli.tsx` | Arg parsing, startup screen, OAuth check, REPL mount |
| REPL | `screens/REPL.tsx` | Interactive loop, message list, input, keyboard shortcuts |
| Query loop | `query.ts` | Sends messages to Claude, dispatches tool calls, recurses |
| Claude API | `services/claude.ts` | SDK wrapper, model selection, retry, prompt caching, cost |
| Context | `context.ts` | Assembles system prompt (git status, CLAUDE.md, README, dir tree) |
| Tools | `tools/*/` | Implements all capabilities Claude can invoke |
| Commands | `commands.ts` | Slash commands (`/help`, `/config`, `/bug`, etc.) |
| Permissions | `permissions.ts` | Per-tool permission gating and storage |
| History | `history.ts` | Persists last 100 CLI commands per project |
| Config | `utils/config.ts` | Reads/writes global and per-project config JSON |
| OAuth | `services/oauth.ts` | PKCE-based authentication with Anthropic Console |
| MCP | `services/mcpClient.ts`, `entrypoints/mcp.ts` | Custom tool protocol (client + server) |
| Telemetry | `services/statsig.ts`, `services/sentry.ts` | Feature flags, error reporting |

---

## Technology Choices

| Concern | Choice |
|---------|--------|
| Terminal UI | React + [Ink](https://github.com/vadimdemedes/ink) |
| AI API | `@anthropic-ai/sdk` (vendored in `vendor/sdk/`) |
| Schema validation | Zod |
| Async patterns | Async generators (`for await`) for streaming messages |
| Bundling | Single `cli.mjs` bundle (not rebuilt from this source) |
| Alt cloud backends | AWS Bedrock, Google Vertex (env-flag selected) |

---

## Data Flow: One Turn

```
User types a message
  → REPL appends UserMessage to messages[]
  → query() called with full message history
    → formatSystemPromptWithContext() builds system prompt
    → querySonnet() calls Claude API (streaming)
    → AssistantMessage yielded → REPL renders it
    → For each tool_use block in response:
        → validate input (Zod)
        → check permissions (hasPermissionsToUseTool)
        → run tool (concurrently if read-only, else serially)
        → yield ProgressMessage while running
        → collect ToolResult
    → If any tool was called:
        → append tool results as UserMessage
        → recurse: call query() again
    → If no more tool calls:
        → generator returns
  → REPL updates cost/token display
```

---

## Concurrency Model

- **Read-only tools** (`isReadOnly() === true`): run concurrently, capped at `MAX_TOOL_USE_CONCURRENCY = 10`.
- **Write tools**: run serially, one at a time.
- Concurrency is managed inside `query.ts` using `Promise.all` batching with `all()` from `utils/generators.ts`.
- An `AbortController` signal threads through the entire call stack; any interrupt (Ctrl-C) cancels in-flight tool calls.
