# Permission System

Claude Code requires explicit user approval before performing potentially destructive operations. This document describes how permissions work, how they are stored, and the special handling for bash commands.

---

## Overview

Every tool has a `needsPermissions(input)` method. When it returns `true`, the REPL pauses and shows a permission dialog before the tool runs. The user can:

- **Allow once** — runs this time only.
- **Allow always** — persists to `projectConfig.allowedTools`.
- **Deny** — the tool receives an error result; Claude sees the denial.

If `dangerouslySkipPermissions=true` (CLI flag), all permission checks are bypassed.

---

## Permission Storage (`src/permissions.ts`)

Granted permissions are stored as strings in `projectConfig.allowedTools` (`.claude/project.json`):

```json
{
  "allowedTools": [
    "Edit",
    "Bash(git:*)",
    "Bash(npm install)",
    "Glob",
    "Grep"
  ]
}
```

### Format

| String | Meaning |
|--------|---------|
| `"Edit"` | FileEditTool allowed for any file |
| `"Bash(git:*)"` | All git subcommands allowed |
| `"Bash(npm install)"` | Exactly `npm install` allowed |
| `"Glob"` | GlobTool always allowed |

### Session-Only Permissions

File-writing tools (`FileEditTool`, `FileWriteTool`, `NotebookEditTool`) store permissions **only in memory** via `grantWritePermissionForOriginalDir()`. These are not persisted and must be re-granted after a restart. This is intentional — write access to the filesystem is considered more sensitive than read access.

---

## BashTool Permission Model

BashTool has the most complex permission model because a single "allowed" decision can cover broad or narrow command sets.

### Command Splitting

Before checking permissions, a bash command is split into subcommands on `||`, `&&`, and `;` operators. Each subcommand is checked independently.

```bash
git status && npm install && rm -rf /
# → ["git status", "npm install", "rm -rf /"]
# Only passes if ALL three subcommands are permitted
```

### Safe Commands (No Approval Needed)

These commands never require user approval:

```
git status, git diff, git log, git show, git branch,
pwd, tree, date, which, echo, cat (on specific files),
ls, node --version, npm --version
```

### Prefix Matching

Stored permissions are matched by prefix:

- `Bash(git:*)` allows `git commit`, `git push`, `git log --oneline`, etc.
- `Bash(npm run:*)` allows `npm run build`, `npm run test`, etc.
- `Bash(npm install)` allows exactly `npm install` (no extra args).

### Command Injection Detection

For commands that look like they might have a computed prefix (e.g., `$(cat file) install`), Claude Code queries a local service (`getCommandSubcommandPrefix()`) to determine the canonical prefix. If this call fails, the permission check **fails closed** — the user is prompted regardless.

### Blanket BashTool Permission

Granting `"Bash"` (without a command qualifier) allows all bash commands without further prompts. This is equivalent to `dangerouslySkipPermissions` for bash.

---

## Banned Commands

Regardless of granted permissions, the following patterns are always blocked in BashTool:

| Command | Reason |
|---------|--------|
| `rm -rf /`, `rm -rf *` | Destructive filesystem operations |
| `git push --force` | Destructive git history rewrite |
| `git reset --hard` | Destructive working directory reset |
| `sudo` | Privilege escalation |
| `chmod -R 777` | Broad permission changes |
| `dd if=...` | Low-level disk writes |

If Claude generates one of these commands, the tool result returns an error explaining why it was blocked.

---

## Permission Check Flow

```
hasPermissionsToUseTool(tool, input, projectConfig, canUseTool)
  │
  ├─ dangerouslySkipPermissions? → allow
  │
  ├─ tool.needsPermissions(input) === false? → allow
  │
  ├─ allowedTools contains matching entry? → allow
  │
  └─ else → call canUseTool callback
              │
              ├─ Shows permission dialog in REPL
              ├─ User approves → optionally save to allowedTools → allow
              └─ User denies → return denial error result to Claude
```

The `canUseTool` callback is provided by the REPL (`src/hooks/useCanUseTool.ts`) and renders the interactive dialog. During non-interactive runs (piped input, `--print` mode), it auto-denies.

---

## MCP Server Permissions

MCP servers loaded from `.mcprc` files (per-directory config) require explicit approval before their tools are loaded. This prevents malicious `.mcprc` files from silently adding tools. The approval state is stored in `approvedMcprcServers` / `rejectedMcprcServers` in the project config.
