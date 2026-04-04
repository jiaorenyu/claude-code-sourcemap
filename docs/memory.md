# Memory and State Management

Claude Code has several distinct layers of "memory" — from ephemeral conversation context to persisted files on disk. This document describes each layer, where it lives, and how it works.

---

## Memory Layers at a Glance

| Layer | Scope | Persistence | Storage |
|-------|-------|-------------|---------|
| System prompt context | Conversation | Ephemeral | RAM (memoized per session) |
| Message history | Conversation | Ephemeral | RAM (messages[] array) |
| Memory files (`MemoryReadTool`/`MemoryWriteTool`) | Cross-session | Persistent | `$MEMORY_DIR` on disk |
| Command history | Project | Persistent | `.claude/project.json` |
| Granted permissions | Project | Persistent | `.claude/project.json` |
| Project config context | Project | Persistent | `.claude/project.json` |
| Global config | User | Persistent | `~/.claude/config.json` |

---

## 1. In-Session Context (Ephemeral)

At the start of each conversation, `context.ts` assembles the system prompt **once** and memoizes it. All recursive turns within the same conversation reuse this cached prompt — context does not refresh mid-conversation.

Sources gathered into the system prompt:

- Git branch, last 5 commits, current diff (snapshot at session start)
- Directory tree (output of `ls`)
- All `**/CLAUDE.md` files discovered via ripgrep
- `README.md`
- User-set key-value pairs from `projectConfig.context`
- Inferred code style

> **Implication**: If you commit new code or change a CLAUDE.md mid-session, Claude won't see it until the next session.

---

## 2. Conversation Message History (Ephemeral)

The REPL maintains a `messages: Message[]` array in React state. Every user message, assistant response, tool call, and tool result is appended here.

This array is passed to `normalizeMessagesForAPI()` before each API call, which:
- Collapses multiple tool results into a single `user` message for the API.
- Strips fields that the API doesn't accept.
- Preserves the full interleaved `user`/`assistant` turn structure required by Claude.

The conversation is **not automatically persisted to disk** between restarts (unless the user uses `/compact` to summarize it).

---

## 3. Memory Files (`MemoryReadTool` / `MemoryWriteTool`)

These tools allow Claude to read from and write to persistent files between sessions. They are **currently gated behind a Statsig feature flag** and disabled for general users.

### Storage Location

```
$MEMORY_DIR/          # Configurable via environment variable
├── index.md          # Convention: table of contents
├── notes.md
└── ...               # Any files Claude writes
```

### MemoryReadTool

- **Without `file_path`**: lists all files in `$MEMORY_DIR` and returns `index.md` content.
- **With `file_path`**: returns the content of that specific file.
- **Security**: validates that the resolved path starts with `$MEMORY_DIR` to prevent path traversal.

### MemoryWriteTool

- Writes (or overwrites) a file at `file_path` inside `$MEMORY_DIR`.
- Creates parent directories automatically (`mkdirSync` with `recursive: true`).
- Returns a simple `"Saved"` confirmation to Claude.
- No deletion capability — files accumulate unless manually removed.

### Intended Usage Pattern

Claude is expected to maintain an `index.md` as a table of contents and write structured notes to individual files. On future sessions, it reads `index.md` first to orient itself, then fetches specific files as needed.

---

## 4. Command History (Per-Project)

`src/history.ts` stores the last 100 CLI commands (the prompts the user typed) in `projectConfig.history`.

```typescript
// Simplified implementation
function addToHistory(command: string) {
  if (history[0] === command) return  // deduplicate
  history.unshift(command)
  history = history.slice(0, 100)
  saveCurrentProjectConfig()
}
```

- Newest command at index 0.
- Deduplication: repeated consecutive commands are not re-added.
- Used for shell-history-style recall in the REPL input.

---

## 5. Granted Permissions (Per-Project)

When a user grants a tool permission, it is saved to `projectConfig.allowedTools`:

```json
{
  "allowedTools": [
    "Edit",
    "Bash(git:*)",
    "Bash(npm install)"
  ]
}
```

File-editing tools (`FileEditTool`, `FileWriteTool`, `NotebookEditTool`) store permissions **in memory only** (not persisted) via `grantWritePermissionForOriginalDir()`. This means file write permissions must be re-granted after a restart.

---

## 6. Project Configuration Context

Users can set arbitrary key-value pairs that are injected into every system prompt:

```
/config context set db_url postgres://localhost/mydb
```

These are stored in `projectConfig.context` and included in the system prompt as:

```xml
<context name="db_url">
  postgres://localhost/mydb
</context>
```

---

## 7. Global User Config (`~/.claude/config.json`)

Stores user-level preferences that apply across all projects:

```typescript
{
  numStartups: number              // How many times Claude Code has been launched
  theme: 'dark' | 'light'
  hasCompletedOnboarding: boolean
  verbose: boolean
  primaryApiKey: string            // Stored API key (if using direct API)
  oauthAccount: {                  // If using OAuth
    accountUuid: string
    emailAddress: string
    organizationUuid: string
  }
  autoUpdaterStatus: string
  customApiKeyResponses: {
    approved: string[]
    rejected: string[]
  }
}
```

---

## 8. Per-Project Config (`.claude/project.json`)

Stored in a `.claude/` directory at the project root:

```typescript
{
  allowedTools: string[]
  context: Record<string, string>
  history: string[]
  mcpServers: Record<string, McpServerConfig>
  approvedMcprcServers: string[]
  rejectedMcprcServers: string[]
  dontCrawlDirectory: boolean       // Auto-set true in home directory
  enableArchitectTool: boolean
  hasTrustDialogAccepted: boolean
  hasCompletedProjectOnboarding: boolean
}
```

---

## Summary: What Survives a Restart

| What | Survives restart? |
|------|-----------------|
| Conversation messages | No |
| System prompt context (git, CLAUDE.md) | Re-gathered fresh each session |
| Memory files (`$MEMORY_DIR`) | Yes (if tools enabled) |
| Granted tool permissions (BashTool, GlobTool, etc.) | Yes (in `allowedTools`) |
| Granted file write permissions | No (session-only) |
| Command history | Yes |
| User-set context (`/config context`) | Yes |
| MCP server configs | Yes |
