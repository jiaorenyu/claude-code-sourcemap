# Configuration

Claude Code uses two JSON config files: one for the user (global) and one per project.

---

## Global Config (`~/.claude/config.json`)

Applies to all projects for the current user.

```typescript
{
  numStartups: number                   // Launch count (used for onboarding gating)
  theme: 'dark' | 'light'
  hasCompletedOnboarding: boolean
  lastOnboardingVersion: string
  verbose: boolean                      // Show extra debug output in REPL
  primaryApiKey: string                 // Direct API key (if not using OAuth)
  oauthAccount: {
    accountUuid: string
    emailAddress: string
    organizationUuid: string
  }
  autoUpdaterStatus: 'disabled' | 'enabled' | 'no_permissions' | 'not_configured'
  customApiKeyResponses: {
    approved: string[]                  // API keys approved by user
    rejected: string[]
  }
  mcpServers: Record<string, McpServerConfig>   // User-level MCP servers
}
```

---

## Project Config (`.claude/project.json`)

Stored at `<project-root>/.claude/project.json`. Project-specific settings and permissions.

```typescript
{
  // Permissions
  allowedTools: string[]                // e.g. ["Edit", "Bash(git:*)"]

  // Injected context
  context: Record<string, string>       // User-set key-value pairs → system prompt

  // Shell history
  history: string[]                     // Last 100 user prompts (newest first)

  // MCP servers
  mcpServers: Record<string, McpServerConfig>
  approvedMcprcServers: string[]        // .mcprc servers user has approved
  rejectedMcprcServers: string[]        // .mcprc servers user has rejected

  // Behaviour flags
  dontCrawlDirectory: boolean           // Skip directory tree in system prompt
                                        // Auto-set to true in home directory
  enableArchitectTool: boolean          // Enable the ArchitectTool
  hasTrustDialogAccepted: boolean
  hasCompletedProjectOnboarding: boolean
}
```

---

## MCP Server Config Shape

```typescript
type McpServerConfig =
  | {
      type?: 'stdio'
      command: string
      args: string[]
      env?: Record<string, string>
    }
  | {
      type: 'sse'
      url: string
    }
```

---

## Environment Variables

| Variable | Effect |
|----------|--------|
| `ANTHROPIC_API_KEY` | API key override (bypasses OAuth) |
| `CLAUDE_CODE_USE_BEDROCK=1` | Route API calls through AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX=1` | Route API calls through Google Vertex |
| `DISABLE_PROMPT_CACHING=1` | Disable prompt cache headers |
| `API_TIMEOUT_MS` | Override default 60s API timeout |
| `MEMORY_DIR` | Directory for MemoryReadTool/MemoryWriteTool files |
| `USER_TYPE=ant` | Enables Anthropic-internal tools and binary feedback |
| `USER_TYPE=SWE_BENCH` | Enables SWE-bench test mode (100 retries, overload retry) |
| `VERTEX_REGION_CLAUDE_3_5_SONNET` | Vertex region override per model |

---

## Slash Command Configuration

Some settings are accessible interactively:

```
/config context set <key> <value>    # Add key-value to system prompt context
/config context delete <key>         # Remove a context key
/config context show                 # List all context keys
/config theme dark|light             # Change terminal theme
```

Context values set this way are stored in `projectConfig.context` and survive restarts.

---

## CLAUDE.md Files

Claude Code automatically searches for `CLAUDE.md` files anywhere in the project tree (using ripgrep with `**/CLAUDE.md`). Their contents are injected into the system prompt at the start of each conversation.

This is the primary mechanism for giving Claude project-specific instructions — coding standards, architecture notes, preferred patterns, commands to run, etc.

**Discovery rules:**
- Searched from the current working directory downward.
- 3-second timeout on the ripgrep search (fails silently if slow).
- All found files are included (not just the root one).
- Home directory (`~`) is excluded from directory crawling; if the user launches Claude from `~`, only an explicit `~/CLAUDE.md` would be found.
