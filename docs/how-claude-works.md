# How Claude Works

This document explains the mechanics of Claude's query loop, how it processes requests, uses tools, and what goes into every API call.

---

## The Query Loop (`src/query.ts`)

`query()` is an **async generator** — it `yield`s `Message` objects as they are produced, so the UI can stream updates in real time.

### High-Level Steps

```
query(messages, options)
  1. Build normalized message list (normalizeMessagesForAPI)
  2. Call querySonnet → stream AssistantMessage
  3. Yield AssistantMessage to REPL
  4. If response contains tool_use blocks:
       For each block:
         a. Parse & validate input (Zod)
         b. Check permissions (canUseTool callback)
         c. Run tool (async generator, yields progress)
         d. Collect ToolResult
  5. Append tool results as a new UserMessage
  6. Recurse: call query() again with updated messages
  7. Repeat until no tool_use in response
```

### Message Types

```typescript
UserMessage      // User input or tool results sent back to Claude
AssistantMessage // Claude's response (text + optional tool_use blocks)
ProgressMessage  // Emitted while a tool is running (shows live status)
```

### Interrupt Handling

An `AbortController` signal is threaded through `query()` and all tool calls. When the user presses **Ctrl-C**:
- The signal fires, cancelling in-flight tool calls.
- The loop yields an `INTERRUPT_MESSAGE` and exits.
- The REPL shows the interrupt in the message history.

---

## System Prompt Assembly (`src/context.ts`)

Before the first API call, `formatSystemPromptWithContext()` constructs the system prompt by gathering several context sources. This is **memoized per conversation** — computed once, not re-gathered on recursive calls.

### Context Sources (in order)

| Source | Description |
|--------|-------------|
| Base instructions | Static prompt defining Claude's role and capabilities |
| Project context | Key-value pairs set by the user via `/config context` |
| Git status | Branch, last 5 commits, recent changes (snapshot at start, never updated) |
| Directory tree | Output of `ls` tool, skipped if `dontCrawlDirectory=true` |
| CLAUDE.md files | All `**/CLAUDE.md` files found via ripgrep (3s timeout) |
| README.md | Project README if present |
| Code style | Inferred from project files |

### Format

Each context source is wrapped in an XML block in the system prompt:

```xml
<context name="git_status">
  On branch main
  Last 5 commits: ...
</context>

<context name="claude_md">
  Contents of ./CLAUDE.md ...
</context>
```

Prompt caching is applied to stable sections (CLAUDE.md, README) to reduce cost on repeated turns.

---

## API Calls (`src/services/claude.ts`)

### Model Selection

| Model | Used for |
|-------|----------|
| Claude 3.7 Sonnet | Primary — all main conversations |
| Claude 3.5 Haiku | Fast queries — tool descriptions, commit message generation |

Alternative backends (AWS Bedrock, Google Vertex) are selected via environment variables:
- `CLAUDE_CODE_USE_BEDROCK=1` → routes through `@anthropic-ai/bedrock-sdk`
- `CLAUDE_CODE_USE_VERTEX=1` → routes through `@anthropic-ai/vertex-sdk`

### Retry Logic

Transient failures are retried with exponential backoff:

```
delay = min(BASE_DELAY_MS × 2^(attempt-1), 32000)
max attempts = 10  (100 in SWE-bench test mode)
```

| Condition | Retried? |
|-----------|----------|
| Connection errors | Yes |
| HTTP 408 (timeout) | Yes |
| HTTP 409 (lock timeout) | Yes |
| HTTP 429 (rate limit) | Yes, respects `retry-after` header |
| HTTP 5xx | Yes |
| HTTP 4xx (other) | No |
| `x-should-retry: false` header | No |
| Overloaded errors | Only in SWE-bench mode |

### Prompt Caching

Enabled by default (disable with `DISABLE_PROMPT_CACHING=1`). Stable content (system prompt, CLAUDE.md, README) is marked with `cache_control: { type: 'ephemeral' }` to reduce input token costs on follow-up turns.

Cache token costs are tracked separately from regular tokens:
- Sonnet cache write: $3.75/M tokens; cache read: $0.30/M tokens
- Haiku cache write: $1.00/M tokens; cache read: $0.08/M tokens

### Cost Tracking

Every API response contributes to a global running total in `cost-tracker.ts`. The REPL displays the accumulated cost in the status bar. Token pricing constants are defined in `src/constants/pricing.ts`.

---

## Tool Execution (`src/tools/*/`)

### Tool Interface

Every tool implements the `Tool<TInput>` interface (`src/Tool.ts`):

```typescript
interface Tool<TInput> {
  name: string
  description(opts?): Promise<string>    // Shown to Claude in the API call
  prompt(opts?): Promise<string>         // Detailed instructions for Claude
  inputSchema: ZodSchema<TInput>
  isEnabled(): Promise<boolean>
  isReadOnly(): boolean                  // Controls concurrency
  needsPermissions(input: TInput): boolean
  validateInput(input, ctx): Promise<ValidationResult>
  call(input, ctx, canUseTool): AsyncGenerator<ToolResult>
  renderToolUseMessage(input): ReactNode
  renderToolResultMessage(output): ReactNode
  renderToolUseRejectedMessage(input): ReactNode
  renderResultForAssistant(output): string
}
```

### Tool Descriptions sent to Claude

`description()` and `prompt()` return strings that are included in the `tools` array of the API request. Claude reads these to decide which tool to call and how to format its input.

### Input Validation

Before a tool runs, its `inputSchema` (a Zod schema) validates the raw JSON input from Claude's `tool_use` block. If validation fails, a `tool_result` with `is_error: true` is returned to Claude — no exception propagates upward.

### Output Truncation

Tool result content is truncated if very large — the first 5,000 characters and last 5,000 characters are preserved, with a truncation notice in between (max 10,000 chars total). This keeps token usage manageable while preserving context at both ends.

---

## Slash Commands (`src/commands.ts`)

Commands are invoked when the user types `/name` in the REPL. Three types exist:

| Type | Behavior |
|------|----------|
| `PromptCommand` | Formats a prompt string and sends it through the normal query loop |
| `LocalCommand` | Runs locally, returns a string rendered directly in the REPL |
| `LocalJSXCommand` | Runs locally, returns a React/Ink component rendered in the REPL |

### Available Commands

`/help`, `/bug`, `/config`, `/cost`, `/clear`, `/compact`, `/init`, `/doctor`, `/login`, `/logout`, `/pr-comments`, `/release-notes`, `/review`, `/terminal-setup`

Internal-only commands (require `USER_TYPE=ant`): `/ctx_viz`, `/resume`, `/listen`

---

## Binary Feedback (Anthropic-Internal)

When `USER_TYPE === 'ant'` and a Statsig gate is enabled, some responses go through a **binary feedback** path:
1. Claude is queried **twice** independently for the same prompt.
2. Both responses are shown side-by-side in the terminal.
3. The user selects the better one (or neither).
4. The selection is logged for model improvement.

This path is completely inactive for external users.
