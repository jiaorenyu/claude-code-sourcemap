# Tools Reference

Claude Code exposes a set of tools to the Claude model. Each tool is an async generator that yields results and renders its own UI in the terminal.

---

## Tool Interface

All tools implement `Tool<TInput>` from `src/Tool.ts`:

```typescript
interface Tool<TInput> {
  name: string
  description(opts?): Promise<string>          // Sent to Claude API
  prompt(opts?): Promise<string>               // Detailed usage instructions
  inputSchema: ZodSchema<TInput>               // Input validation
  isEnabled(): Promise<boolean>                // Runtime on/off switch
  isReadOnly(): boolean                        // true → can run concurrently
  needsPermissions(input): boolean             // false → no approval needed
  validateInput(input, ctx): Promise<ValidationResult>
  call(input, ctx, canUseTool): AsyncGenerator<ToolResult>
  renderToolUseMessage(input): ReactNode       // Shows what Claude requested
  renderToolResultMessage(output): ReactNode   // Shows the result
  renderToolUseRejectedMessage(input): ReactNode
  renderResultForAssistant(output): string     // Formats result back to Claude
}
```

### Concurrency

- `isReadOnly() === true` → tool runs concurrently with other read-only tools (up to 10 at a time).
- `isReadOnly() === false` → tool runs serially; no other write tools run in parallel.

---

## Available Tools

### BashTool

**Executes shell commands** in a persistent shell that maintains working directory state across calls.

- **Read-only**: No
- **Permissions**: Required. Checked per-command with prefix-based matching (e.g., `Bash(git:*)` allows all git commands).
- **Timeout**: Configurable per call, max 600 seconds (600,000ms).
- **Banned commands**: `rm`, `rmdir`, `git push --force`, `git reset --hard`, `sudo`, and several other destructive commands.
- **cd restrictions**: Can only navigate to subdirectories of the original working directory.
- **Output**: Stdout and stderr combined; stderr wrapped in `<error>` tags for Claude.
- **Truncation**: Very large outputs are truncated to first+last 5k characters.

Safe commands that don't require user approval: `git status`, `git diff`, `git log`, `pwd`, `tree`, `date`, `which`.

---

### FileReadTool

**Reads file content**, optionally limited to a line range.

- **Read-only**: Yes
- **Permissions**: Not required.
- **Input**: `file_path`, optional `offset` (start line) and `limit` (line count).
- **Output**: File content with line numbers in `cat -n` format.
- **Encoding**: Auto-detects file encoding.
- **Images**: Can read images (PNG, JPG, etc.) — returns a content block the model can interpret visually.

---

### FileEditTool

**Edits a file** by replacing an exact string with a new string.

- **Read-only**: No
- **Permissions**: Session-only (must be granted once per session, not persisted).
- **Operations**:
  - `old_string=""` → creates a new file (fails if it already exists)
  - `new_string=""` → deletes the matched section
  - Both non-empty → standard string replacement
- **Validation**: `old_string` must appear exactly once in the file (no ambiguous matches).
- **Output**: Renders a diff with 4 lines of context via `StructuredDiff` component.
- **Encoding**: Detects and preserves original file encoding and line endings.

---

### FileWriteTool

**Writes (or overwrites) a file** with new content.

- **Read-only**: No
- **Permissions**: Session-only.
- **Use case**: Creating new files or doing complete rewrites where diff-style editing is impractical.

---

### GlobTool

**Finds files by glob pattern** (e.g., `src/**/*.ts`).

- **Read-only**: Yes
- **Permissions**: Not required.
- **Output**: Matching file paths sorted by modification time (most recent first).
- **Backend**: Uses the same glob library as the bundled Ripgrep.

---

### GrepTool

**Searches file content** using regular expressions.

- **Read-only**: Yes
- **Permissions**: Not required.
- **Input**: Pattern (regex), optional `path` and `include` (glob filter).
- **Backend**: Ripgrep for fast searching.
- **Output**: Matching lines with file path and line number.

---

### lsTool

**Lists directory contents**, optionally filtered.

- **Read-only**: Yes
- **Permissions**: Not required.
- **Output**: Formatted directory tree. Used internally for the context directory snapshot.

---

### AgentTool

**Spawns a sub-agent** — a new Claude instance with its own conversation context and tool access.

- **Read-only**: No
- **Permissions**: Required.
- **Input**: A `prompt` describing the task for the sub-agent.
- **Use case**: Parallelizable subtasks, isolated investigations, long-running research that would pollute the main conversation.
- **Context**: The sub-agent has its own message history but inherits the system prompt (context, CLAUDE.md, etc.) from the parent.

---

### ArchitectTool

**Generates architectural plans or design documents** using a separate Claude call with extended thinking.

- **Read-only**: Yes
- **Permissions**: Not required.
- **Gated**: Only enabled when `projectConfig.enableArchitectTool === true`.

---

### ThinkTool

**Extended thinking** — allows Claude to reason through a problem in a scratchpad before responding.

- **Read-only**: Yes
- **Permissions**: Not required.
- **Gated**: Behind a Statsig feature flag.
- **Input**: A `thought` string — Claude writes its reasoning here.
- **Output**: The thought is not returned to Claude as tool output; it's purely a reasoning aid.

---

### NotebookReadTool

**Reads a Jupyter notebook** (`.ipynb` file), returning cells and their outputs.

- **Read-only**: Yes
- **Permissions**: Not required.

---

### NotebookEditTool

**Edits a Jupyter notebook cell** by replacing its source.

- **Read-only**: No
- **Permissions**: Session-only.
- **Input**: `notebook_path`, `cell_id` (cell index), `new_source`.

---

### MCPTool

A **proxy tool** that wraps tools exposed by connected MCP servers. Each MCP server tool becomes its own `MCPTool` instance with the server's name, input schema, and description. See [mcp.md](./mcp.md).

---

### MemoryReadTool / MemoryWriteTool

**Persistent memory** — reads from and writes to files in `$MEMORY_DIR`.

- Currently **disabled** for general users (behind Statsig gate).
- Intended for cross-session note-taking.
- See [memory.md](./memory.md) for details.

---

### StickerRequestTool

Requests Anthropic stickers. Anthropic-internal only (`USER_TYPE === 'ant'`).

---

## Tool Output Format

Tool results are always returned to Claude as `tool_result` content blocks:

```typescript
{
  type: 'tool_result',
  tool_use_id: string,
  content: string | ContentBlock[],
  is_error?: boolean
}
```

If `is_error: true`, Claude sees this as a failure and typically tries to recover or explains the error to the user.

Output longer than ~10,000 characters is truncated: the first 5,000 and last 5,000 characters are preserved with a truncation notice in between.
