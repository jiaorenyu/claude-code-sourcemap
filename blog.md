# Inside Claude Code: How Anthropic Built an AI Coding Agent

When Anthropic published `@anthropic-ai/claude-code` to npm and then quietly deleted the package, they accidentally left something behind: source maps embedded in the compiled `cli.mjs` bundle. Source maps are normally a debugging convenience — they map minified code back to its original TypeScript. In this case, they mapped back to the full, original source.

The recovered source for v0.2.8 is [now on GitHub](https://github.com/jiaorenyu/claude-code-sourcemap). I spent time reading through it and was surprised by how much thoughtful engineering is packed into what looks, from the outside, like a simple terminal chatbot. This post covers what I found.

---

## The Basic Architecture

Claude Code is a React application — but instead of running in a browser, it runs in your terminal. It uses a library called [Ink](https://github.com/vadimdemedes/ink) that implements a React renderer for the terminal. Your `<Box>` and `<Text>` components become ANSI escape codes.

The entry point is `src/entrypoints/cli.tsx`. It parses arguments, checks for OAuth credentials, and mounts a single React component called `REPL` (`src/screens/REPL.tsx`), which manages the interactive session from there.

The actual Claude interaction flows like this:

```
User types a message
  → REPL appends it to a messages[] array
  → query() is called with the full message history
    → System prompt is assembled (git status, CLAUDE.md files, directory tree)
    → Claude API is called (streaming)
    → Response is yielded back to REPL as it arrives
    → If Claude requested tool calls:
        → Tools run (concurrently if read-only, serially if write)
        → Results appended as a new user message
        → query() is called again (recursion)
    → Repeat until Claude stops calling tools
```

The `query()` function in `src/query.ts` is an **async generator**. It `yield`s message objects — user messages, assistant messages, progress updates — as they're produced. This is what lets the REPL stream text and show live tool progress without any special callback machinery.

---

## What Claude Sees Before You Type Anything

Before the first API call, `src/context.ts` assembles the system prompt. This is memoized — computed once at session start and reused for every recursive call within the conversation.

Here's what goes in:

1. **Base instructions** — A static prompt defining Claude's role and capabilities.
2. **User-set context** — Key-value pairs you've set via `/config context set key value`.
3. **Git status** — Current branch and last 5 commits. This is a snapshot; it won't update if you commit something mid-session.
4. **Directory tree** — Output from the `ls` tool, giving Claude a map of the project layout.
5. **All `CLAUDE.md` files** — Discovered via ripgrep with a 3-second timeout. Every `CLAUDE.md` anywhere in the project tree is included.
6. **`README.md`** — The project readme, if present.
7. **Inferred code style** — Derived from project files.

Each source is wrapped in XML tags in the system prompt:

```xml
<context name="git_status">
  On branch main
  Last 5 commits: ...
</context>

<context name="claude_md">
  # My Project
  Always use TypeScript strict mode...
</context>
```

The `CLAUDE.md` discovery is worth emphasizing: it's not just the root `CLAUDE.md`. It searches `**/CLAUDE.md` — so you can have per-directory instructions and Claude will find all of them. This is how teams can put domain-specific instructions close to the code they describe.

---

## The Tool System

Claude Code gives Claude 16 tools to work with. Each tool lives in `src/tools/<ToolName>/` and implements a standard interface:

```typescript
interface Tool<TInput> {
  name: string
  description(): Promise<string>        // What Claude reads to decide when to use this
  inputSchema: ZodSchema<TInput>        // Input validation
  isReadOnly(): boolean                 // Affects concurrency
  needsPermissions(input): boolean
  call(input, ctx): AsyncGenerator      // Actual execution
  renderToolUseMessage(input): ReactNode
  renderToolResultMessage(output): ReactNode
}
```

The `isReadOnly()` flag is how concurrency is managed. When Claude calls multiple tools simultaneously (it does this often for parallel file reads), read-only tools run concurrently — up to 10 at a time. Write tools run serially. This is enforced in `query.ts` with a cap of `MAX_TOOL_USE_CONCURRENCY = 10`.

The full tool list:

| Tool | What it does |
|------|-------------|
| `BashTool` | Executes shell commands in a persistent shell |
| `FileReadTool` | Reads files, supports line ranges and images |
| `FileEditTool` | Edits files via exact string replacement |
| `FileWriteTool` | Creates or overwrites files |
| `GlobTool` | Finds files by pattern |
| `GrepTool` | Searches file content with regex (backed by ripgrep) |
| `lsTool` | Lists directory contents |
| `AgentTool` | Spawns a sub-agent with its own conversation |
| `ArchitectTool` | Generates architectural plans (extended thinking) |
| `ThinkTool` | Scratchpad for Claude's reasoning (not returned as output) |
| `NotebookReadTool` | Reads Jupyter notebooks |
| `NotebookEditTool` | Edits notebook cells |
| `MCPTool` | Proxies tools from connected MCP servers |
| `MemoryReadTool` | Reads from persistent memory files |
| `MemoryWriteTool` | Writes to persistent memory files |
| `StickerRequestTool` | Requests Anthropic stickers (internal only) |

### BashTool Is More Cautious Than You'd Think

BashTool doesn't just execute commands. Before running anything:

1. The command is split on `||`, `&&`, and `;` into subcommands.
2. Each subcommand is checked independently against stored permissions.
3. A separate service call (`getCommandSubcommandPrefix()`) detects whether a command has a computed prefix (e.g., a `$(...)` substitution). If this call fails, the check **fails closed** — the user is always prompted.

Some commands are permanently banned regardless of permissions: `rm -rf`, `git push --force`, `git reset --hard`, `sudo`, and a few others.

Permission strings are stored as prefix patterns in the project config:
- `"Bash(git:*)"` — allows all git subcommands
- `"Bash(npm install)"` — allows exactly `npm install`
- `"Bash"` — allows everything (equivalent to `--dangerously-skip-permissions` for bash)

---

## How Memory Works (All of It)

"Memory" in Claude Code is more nuanced than a single system. There are actually several distinct layers:

### Layer 1: The System Prompt (Ephemeral, Per-Session)

Everything in the system prompt — git status, CLAUDE.md contents, directory tree — exists only for the current conversation. It's gathered once at startup and never refreshed. Start a new session to pick up changes.

### Layer 2: The Message History (Ephemeral, Per-Session)

The `messages[]` array in REPL state holds the full conversation. Every user message, Claude response, tool call, and tool result lives here. It's passed to `normalizeMessagesForAPI()` before each API call, which collapses tool results and strips non-API fields into the format the API expects.

This is not persisted to disk between restarts.

### Layer 3: Memory Files (Persistent, Cross-Session)

The `MemoryReadTool` and `MemoryWriteTool` give Claude a place to write notes that survive across sessions. Files are stored in `$MEMORY_DIR` (configurable via environment variable).

The design is intentionally simple — it's just a directory of files. Claude is expected to maintain an `index.md` as a table of contents and write structured notes to individual files. On future sessions, it reads `index.md` first to know what it wrote before.

These tools are currently **disabled** for general users behind a Statsig feature flag, but the implementation is fully present in the source.

### Layer 4: Command History (Persistent, Per-Project)

`src/history.ts` stores the last 100 prompts you typed, in `.claude/project.json`. Newest first. Deduplicated. This is used for shell-history-style recall in the REPL input. Simple, but useful.

### Layer 5: Project Config (Persistent, Per-Project)

`.claude/project.json` stores:
- Granted tool permissions
- User-set context key-values
- MCP server configs
- Behavioral flags (`dontCrawlDirectory`, `enableArchitectTool`, etc.)

File write permissions are specifically **not** persisted here. `FileEditTool` and `FileWriteTool` store permissions in memory only — you have to re-grant them each session. This is intentional; write access to the filesystem is treated as more sensitive.

---

## The API Layer: Retry, Caching, and Cost

`src/services/claude.ts` is where API calls actually happen. A few things stood out:

### Retry Logic

Failures are retried with exponential backoff up to 10 times:

```
delay = min(BASE_DELAY_MS × 2^(attempt-1), 32 seconds)
```

Rate limits (429) respect the `retry-after` header. 5xx errors are retried. 4xx errors (except 408 timeout and 409 lock) are not. There's a special `USER_TYPE=SWE_BENCH` mode that bumps retries to 100 — presumably for running the SWE-bench benchmark where you really don't want transient failures to invalidate a run.

### Prompt Caching

Prompt caching is on by default. Stable parts of the system prompt — CLAUDE.md contents, README, tool descriptions — are marked with `cache_control: { type: 'ephemeral' }`. On follow-up turns, the API can return a cached version instead of re-processing those tokens, which meaningfully reduces cost on long conversations.

Cache read tokens cost ~10x less than regular input tokens on Sonnet ($0.30/M vs $3.00/M), so the savings on a long session with a big CLAUDE.md can be significant.

### Model Selection

The main model is Claude 3.7 Sonnet. Claude 3.5 Haiku is used for lighter-weight operations — generating commit message suggestions, tool descriptions, and similar tasks where speed matters more than depth.

Both AWS Bedrock and Google Vertex are supported as alternative backends, selected via environment variables.

---

## Sub-Agents: Claude Spawning Claude

`AgentTool` is one of the more interesting tools. It lets Claude spawn a sub-agent — an entirely new Claude instance with its own message history — to handle a subtask. The sub-agent gets the same system prompt (same CLAUDE.md files, same project context) but starts with a clean conversation and a prompt describing what it should do.

This is useful for:
- Tasks that can run in parallel (two sub-agents investigating different files simultaneously)
- Long research tasks that would pollute the main conversation
- Isolated experiments where you don't want the main conversation to see failed attempts

The sub-agent's final response is returned as the tool result to the parent Claude. The parent never sees the sub-agent's internal tool calls or reasoning — just the conclusion.

---

## MCP: The Extensibility Layer

Claude Code supports the [Model Context Protocol](https://modelcontextprotocol.io/) in both directions.

As a **client**, it connects to external MCP servers configured in `.claude/project.json` or `~/.claude/config.json`. Each server's tools become available to Claude alongside the built-in tools. Servers can be local processes (stdio) or HTTP SSE endpoints.

As a **server**, Claude Code can expose its own tools (`BashTool`, `FileEditTool`, etc.) to other MCP clients via `claude --mcp-server`. This lets other tools and agents use Claude Code's file manipulation and shell capabilities.

One security detail worth noting: `.mcprc` files — directory-local MCP configs that could be committed by a third party — require explicit user approval before their tools load. Approved server IDs are persisted in the project config to avoid repeated prompting.

---

## What's Gated and Why

Several features are present in the source but disabled:

| Feature | Gate |
|---------|------|
| `MemoryReadTool` / `MemoryWriteTool` | Statsig feature flag |
| `ThinkTool` | Statsig feature flag |
| Binary feedback UI | Statsig + `USER_TYPE=ant` |
| `StickerRequestTool` | `USER_TYPE=ant` only |
| `ArchitectTool` | `projectConfig.enableArchitectTool=true` |

The binary feedback system is particularly interesting. When enabled for Anthropic-internal users, some responses trigger a mode where Claude is queried **twice** for the same prompt, both responses are shown side-by-side, and the user picks the better one. The selection is logged. This is presumably how Anthropic collects preference data for RLHF without needing to run separate annotation pipelines.

---

## A Few Design Choices Worth Noting

**The async generator pattern is elegant.** Using `async function*` for the query loop means the UI and the execution engine are loosely coupled — the generator yields events, the REPL consumes them, and neither needs to know about the other's internals.

**Output truncation is thoughtful.** When a tool produces very large output, Claude Code doesn't just truncate from the end. It keeps the first 5,000 characters and the last 5,000 characters, with a truncation notice in between. For log output or stack traces, the beginning (what started) and the end (what went wrong) are usually the most relevant parts.

**Permissions fail closed.** When the command injection detection call fails (network error, etc.), Claude Code doesn't assume the command is safe. It prompts the user. This is the right default.

**Home directory protection.** If you launch Claude Code from your home directory (`~`), it sets `dontCrawlDirectory=true` automatically. Without this, the directory tree context would contain your entire filesystem, which would be both huge and a privacy concern.

---

## Reading the Source Yourself

The full extracted source is at [github.com/jiaorenyu/claude-code-sourcemap](https://github.com/jiaorenyu/claude-code-sourcemap). The active fork building on top of this is [dnakov/anon-kode](https://github.com/dnakov/anon-kode).

Start with `src/query.ts` if you want to understand the core loop, `src/context.ts` if you want to understand what Claude knows about your project, and `src/tools/BashTool/BashTool.tsx` if you want to understand how the most complex tool works.

There's more in there — the OAuth flow, the Statsig integration, the cost tracking, the persistent shell implementation — but this covers the parts that matter most for understanding how the system works.
