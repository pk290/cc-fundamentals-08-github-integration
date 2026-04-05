# Claude Code: Features & Concepts Spec Sheet

A reference guide explaining the key building blocks of Claude Code — what each one is, how it differs from the others, and when to use it.

---

## Quick Comparison

| Feature | What It Is | Where It Lives | Who Calls It |
|---|---|---|---|
| **Slash Commands** | Built-in or custom text shortcuts | CLI / `.claude/commands/` | You, manually |
| **Skills** | Reusable prompt templates (custom slash commands) | `.claude/commands/*.md` | You, via `/skill-name` |
| **MCP Servers** | External tool/data integrations via a protocol | Config file / remote | Claude, automatically |
| **Hooks** | Shell scripts that run on Claude Code events | `settings.json` | System, automatically |
| **CLAUDE.md** | Persistent instructions for Claude | Root or any dir | Claude reads on start |

---

## Slash Commands (`/command`)

Slash commands are **short text shortcuts** typed directly into the Claude Code prompt. They trigger specific behaviors or workflows.

### Built-in Commands

| Command | What It Does |
|---|---|
| `/help` | Show available commands and usage |
| `/clear` | Clear conversation context |
| `/compact` | Compress conversation to save context |
| `/cost` | Show token usage and cost for the session |
| `/model` | Switch the active Claude model |
| `/review` | Review recent changes |
| `/commit` | Stage and commit changes with a generated message |
| `/pr` | Create a pull request |
| `/bug` | Report a bug in Claude Code |
| `/doctor` | Check Claude Code installation health |
| `/init` | Initialize Claude Code in a project (creates CLAUDE.md) |
| `/memory` | View or edit Claude's memory files |
| `/settings` | Open settings |
| `/fast` | Toggle fast mode (same model, faster output) |
| `/vim` | Toggle vim keybindings |

### How to Use

```
> /help
> /commit
> /model claude-opus-4-6
```

### Key Point

Built-in slash commands are **hardcoded into Claude Code**. You cannot modify them. To create your own, use **Skills** (see below).

---

## Skills (Custom Slash Commands)

Skills are **user-defined slash commands** — reusable prompt templates stored as Markdown files. When invoked, the file's contents are injected as a prompt into the conversation.

### Where They Live

```
.claude/
└── commands/
    ├── my-skill.md       → invoked as /my-skill
    ├── review-pr.md      → invoked as /review-pr
    └── deploy-check.md   → invoked as /deploy-check
```

### File Format

```markdown
---
name: review-pr
description: Review a pull request for quality, bugs, and security issues
---

Review the current PR diff carefully. Check for:
- Logic bugs and edge cases
- Security vulnerabilities (injection, auth bypass, etc.)
- Performance issues
- Missing tests
- Code style inconsistencies

Provide specific, actionable feedback with file paths and line numbers.
```

### How to Use

```
> /review-pr
> /deploy-check
> /my-skill some additional argument text
```

### Argument Passing

Text after the skill name is appended to the prompt as additional context:

```
> /my-skill focus on the auth module
```

### Scope

- **Project skills**: `.claude/commands/` in your repo — shared with anyone who uses Claude Code in that project
- **Global skills**: `~/.claude/commands/` — available in every project on your machine

### Key Point

Skills are just Markdown files. They don't run code — they inject text into the Claude conversation. For running actual code or accessing external systems, use **MCP Servers**.

---

## MCP Servers (Model Context Protocol)

MCP (Model Context Protocol) is an **open standard** that lets Claude connect to external tools, APIs, and data sources. An MCP server exposes capabilities that Claude can invoke autonomously during a conversation.

### What MCP Servers Provide

| Capability | Description | Example |
|---|---|---|
| **Tools** | Functions Claude can call | `search_web`, `read_file`, `query_db` |
| **Resources** | Data Claude can read | Files, database records, API responses |
| **Prompts** | Reusable prompt templates | Pre-built prompts from the server |

### How It Works

```
Claude ──► MCP Client ──► MCP Server ──► External System
                                         (GitHub, Slack, DB, filesystem...)
```

1. You configure an MCP server in Claude Code's settings
2. Claude sees the server's tools listed as available capabilities
3. During conversation, Claude calls tools as needed (with your permission)
4. Results flow back into the conversation context

### Configuration

MCP servers are configured in `~/.claude/settings.json` (global) or `.claude/settings.json` (project-level):

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token-here"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allow"]
    }
  }
}
```

### Transport Types

| Type | Description | Use Case |
|---|---|---|
| **stdio** | Subprocess communication via stdin/stdout | Local tools, CLIs |
| **SSE** | HTTP Server-Sent Events | Remote/cloud services |

### Popular MCP Servers

- `@modelcontextprotocol/server-filesystem` — Read/write local files
- `@modelcontextprotocol/server-github` — GitHub API (issues, PRs, repos)
- `@modelcontextprotocol/server-brave-search` — Web search
- `@modelcontextprotocol/server-postgres` — PostgreSQL queries
- `@modelcontextprotocol/server-slack` — Slack messages and channels

### Key Point

MCPs extend **what Claude can do** — they give Claude access to real systems and data. Skills extend **what Claude knows to say** — they inject instructions. These are complementary, not alternatives.

---

## Hooks

Hooks are **shell commands that execute automatically** in response to Claude Code lifecycle events. They allow you to enforce policies, run validators, or automate side effects without manual intervention.

### Configured In

`~/.claude/settings.json` or `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run bash command'"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

### Hook Events

| Event | Fires When |
|---|---|
| `PreToolUse` | Before Claude calls any tool |
| `PostToolUse` | After a tool call completes |
| `Notification` | Claude sends a notification |
| `Stop` | Claude finishes a response |

### Hook Exit Codes

| Exit Code | Meaning |
|---|---|
| `0` | Success — proceed normally |
| `2` | Block the tool call (Claude sees the stderr output) |
| Other | Non-blocking error, logged to transcript |

### Common Use Cases

- Auto-format code after every file edit (`PostToolUse` on `Edit`)
- Block dangerous commands (`PreToolUse` on `Bash` — check for `rm -rf`)
- Run tests after changes (`PostToolUse` on `Write`)
- Send desktop notifications when Claude finishes a long task (`Stop`)
- Log all tool usage to a file for auditing

### Key Point

Hooks run **outside of Claude's context** — Claude doesn't see them execute or control them. They are enforced by the Claude Code harness, making them reliable for security and automation policies.

---

## CLAUDE.md

`CLAUDE.md` is a **persistent instruction file** that Claude reads automatically at the start of every session in that directory. It's how you give Claude permanent context about your project.

### Where It Lives

Claude reads `CLAUDE.md` from:
1. The current working directory (project-level)
2. Parent directories (walking up to root)
3. `~/.claude/CLAUDE.md` (global, always loaded)

### What to Put in It

```markdown
# Project: My App

## Stack
- Backend: Python 3.12 + FastAPI
- Frontend: React 18 + TypeScript
- Database: PostgreSQL 15

## Development Commands
- Run tests: `pytest`
- Start dev server: `uvicorn main:app --reload`
- Lint: `ruff check .`

## Conventions
- Use snake_case for Python, camelCase for TypeScript
- All API endpoints require authentication except /health
- Never commit directly to main — always use PRs

## Key Files
- `src/auth/`: Authentication logic
- `src/api/routes/`: All API route handlers
- `tests/`: Pytest test suite
```

### Key Point

CLAUDE.md is read-only from Claude's perspective during a session (Claude won't modify it unless you ask). It's your way of saying "always remember this" without having to repeat it every conversation.

---

## "Plugins" in Claude Code

> **Note:** "Plugins" is not an official Claude Code term. Depending on context, people use it to mean one of the above features.

| When someone says "plugin" they might mean... | What they actually mean |
|---|---|
| Adding a new capability or tool | **MCP Server** |
| A reusable command or prompt | **Skill** |
| A behavior that runs automatically | **Hook** |
| Persistent project configuration | **CLAUDE.md** |

If you encounter "plugin" in Claude Code discussions, ask which of the above they mean.

---

## How They Work Together

A typical advanced Claude Code setup combines all of these:

```
.claude/
├── settings.json          ← MCP servers + Hooks configured here
├── commands/
│   ├── review-pr.md       ← Skill: /review-pr
│   ├── deploy.md          ← Skill: /deploy
│   └── standup.md         ← Skill: /standup
└── CLAUDE.md              ← Always-on project context

~/.claude/
├── settings.json          ← Global MCP servers (e.g., filesystem, web search)
├── commands/
│   └── my-template.md     ← Global skill available everywhere
└── CLAUDE.md              ← Global preferences (tone, style, etc.)
```

**Workflow example:**

1. Claude reads `CLAUDE.md` → knows the stack, conventions, key files
2. You type `/review-pr` → **skill** injects review instructions
3. Claude calls `github` MCP tool → reads actual PR diff from GitHub
4. After Claude edits a file → **hook** auto-runs `prettier`
5. Claude uses built-in `/commit` → stages and commits the fix

---

## Decision Guide: What Should I Use?

| Goal | Use |
|---|---|
| Run a specific workflow repeatedly | **Skill** (`/command`) |
| Give Claude access to an external system | **MCP Server** |
| Enforce a policy or automate side effects | **Hook** |
| Give Claude permanent project context | **CLAUDE.md** |
| Quick one-off actions (clear, cost, model) | **Built-in slash command** |

---

## Further Reading

- [MCP Protocol Specification](https://modelcontextprotocol.io)
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
- [MCP Server Registry](https://github.com/modelcontextprotocol/servers)
