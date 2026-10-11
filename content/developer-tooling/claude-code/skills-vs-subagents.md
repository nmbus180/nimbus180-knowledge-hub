---
title: "Claude Code Skills, Subagents, MCP & Hooks"
description: "How the four Claude Code extension mechanisms differ: a skills vs. subagents comparison, plus detailed descriptions of MCP servers and hooks."
category: developer-tooling
tags:
  - claude-code
  - skills
  - subagents
  - mcp
  - hooks
  - ai-engineering
status: draft
visibility: private
updated: 2026-10-11
---

# Claude Code Skills, Subagents, MCP & Hooks

**Nimbus180 Engineering Knowledge Hub**

Claude Code can be extended in four ways. Each answers a different question:

| Mechanism | Question it answers | Who decides when it runs |
|---|---|---|
| Skill | *How* should this kind of task be done? | The model (or the user, by name) |
| Subagent | *Who* should do this piece of work? | The model (or the user, by name) |
| MCP server | *What* external systems can Claude reach? | The model, by calling a tool |
| Hook | *What must always happen* at a given point? | The harness, deterministically |

## 1. Skills vs. subagents

| | Skill | Subagent |
|---|---|---|
| What it is | Instructions + resources (files/scripts) | An agent instance |
| Context | Loaded into the current context | Runs in a separate context |
| Who does the work | The current agent | A different agent |
| Cost to context | Adds tokens to the main window (only when used) | Main window only sees the final report |
| Parallelism | None | Yes |
| Tool scoping | Uses whatever tools the current agent has | Can restrict or expand tools, MCP servers, model |
| Model | Same model | Can override (Haiku/Sonnet/Opus) |
| Information flow | Full: agent sees all details, intermediate results stay visible | Lossy: only the summary comes back |
| Defined as | `SKILL.md` folder | `AgentDefinition` in code or `.claude/agents/*.md` |
| Analogy | Handing an engineer a runbook | Assigning a ticket to another engineer |

## 2. Model Context Protocol (MCP)

### What it is

MCP is an open protocol that lets an AI application connect to external systems through a standard interface. An **MCP server** is a small program that exposes a system (GitHub, Azure, a database, an internal API) to Claude. Claude Code is the **MCP client**: it connects to the servers you configure and makes what they offer available during a session.

Without MCP, Claude can only work with the local filesystem and shell. With it, Claude can query a database, read a work item, or inspect cloud resources directly instead of asking you to paste the output.

### What a server can expose

| Primitive | What it is | How it appears in Claude Code |
|---|---|---|
| Tools | Functions Claude can call, with typed inputs | Callable tools named `mcp__<server>__<tool>` |
| Resources | Read-only data the server can supply (files, records, schemas) | Referenced with `@` mentions |
| Prompts | Parameterised prompt templates published by the server | Slash commands named `/mcp__<server>__<prompt>` |

Most servers expose only tools.

### Transports

| Transport | How it runs | Typical use |
|---|---|---|
| stdio | Claude Code launches the server as a local process and talks over stdin/stdout | Local tools, CLIs, anything needing access to the developer's machine |
| HTTP | Claude Code connects to a remote URL | Hosted services, usually with OAuth |
| SSE | Older remote transport | Legacy servers only; prefer HTTP |

### Configuration scopes

| Scope | Stored in | Visible to | Use for |
|---|---|---|---|
| Local (default) | User settings, keyed to this project | You, in this project | Personal or experimental servers, anything with your own credentials |
| Project | `.mcp.json` at the repository root (committed) | Everyone who clones the repo | Servers the whole team needs |
| User | User settings | You, in every project | Personal utilities you use everywhere |

Project-scoped servers require each developer to approve them the first time, because `.mcp.json` can launch arbitrary commands.

### Adding and managing servers

```bash
# Remote server over HTTP
claude mcp add --transport http <name> <url>

# Local stdio server (everything after -- is the command to launch)
claude mcp add <name> --env API_KEY=value -- <command> [args...]

# Share with the team by writing to .mcp.json
claude mcp add --scope project <name> -- <command> [args...]

claude mcp list            # show configured servers and status
claude mcp get <name>      # show one server's configuration
claude mcp remove <name>   # remove a server
```

Inside a session, `/mcp` shows connection status and starts the OAuth sign-in flow for servers that need it.

### Example `.mcp.json`

```json
{
  "mcpServers": {
    "work-items": {
      "type": "http",
      "url": "https://mcp.example.com/mcp"
    },
    "local-db": {
      "command": "npx",
      "args": ["-y", "<database-mcp-package>"],
      "env": {
        "CONNECTION_STRING": "${DEV_DB_CONNECTION_STRING}"
      }
    }
  }
}
```

`${VAR}` is expanded from the developer's environment, so the committed file contains no secrets.

### Costs and risks

- **Context cost.** Every connected server's tool definitions take up context. Connect what the project needs, not everything available.
- **Trust.** A stdio server runs with your user's permissions. A remote server sees whatever Claude sends it. Install only servers from sources you trust.
- **Prompt injection.** Content returned by a server (an issue body, a web page, a database row) is untrusted input that may contain instructions. Be most careful with servers that fetch arbitrary external content.
- **Least privilege.** Give servers read-only or narrowly scoped credentials wherever possible, and never point one at production with write access.
- **Permissions.** MCP tools go through the same allow/ask/deny permission rules as built-in tools, and can be allowed per server or per tool.

## 3. Hooks

### What they are

A hook is a shell command that Claude Code runs automatically at a fixed point in its lifecycle. Hooks are executed by the harness, not chosen by the model, so they are **deterministic**: an instruction in `CLAUDE.md` is a request Claude usually follows, while a hook is a rule that always runs.

Use a hook when something must happen every time: formatting after an edit, blocking writes to protected files, logging commands, or injecting context at session start.

### Hook events

| Event | Fires | Can block? | Typical use |
|---|---|---|---|
| `PreToolUse` | Before a tool call runs | Yes, the tool call | Block dangerous commands, protect files |
| `PostToolUse` | After a tool call succeeds | No (the tool already ran), but can send feedback to Claude | Format, lint, run tests on changed files |
| `UserPromptSubmit` | When the user submits a prompt, before Claude sees it | Yes, the prompt | Add context, validate prompts |
| `Notification` | When Claude Code sends a notification (e.g. waiting for permission) | No | Desktop or chat alerts |
| `Stop` | When Claude finishes responding | Yes, forces Claude to continue | Require tests to pass before finishing |
| `SubagentStop` | When a subagent finishes | Yes, forces it to continue | Validate subagent output |
| `PreCompact` | Before the conversation is compacted | No | Save a transcript or notes |
| `SessionStart` | When a session starts or resumes | No | Load project context, set up the environment |
| `SessionEnd` | When a session ends | No | Cleanup, logging |

### Where hooks are configured

| File | Scope |
|---|---|
| `~/.claude/settings.json` | You, in every project |
| `.claude/settings.json` | The project, committed and shared with the team |
| `.claude/settings.local.json` | You, in this project (not committed) |

### Configuration structure

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/protect-secrets.js\""
          }
        ]
      }
    ]
  }
}
```

- The top-level key is the **event**.
- `matcher` selects which tools the hook applies to. It is matched against the tool name and accepts a single name (`Bash`), alternatives (`Edit|Write`), or a regex (`mcp__github__.*`). Omit it to match every tool. It applies only to tool events.
- `hooks` lists the commands to run. `$CLAUDE_PROJECT_DIR` resolves to the project root.

### What a hook receives and returns

The hook receives a JSON object on **stdin** describing the event:

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": { "file_path": "src/appsettings.Production.json", "...": "..." }
}
```

It reports its result through its **exit code**:

| Exit code | Meaning |
|---|---|
| `0` | Success; execution continues. For `UserPromptSubmit` and `SessionStart`, stdout is added to Claude's context. |
| `2` | Blocking error. stderr is sent back to Claude, and the action is blocked where the event allows it (see the table above). |
| Other | Non-blocking error. stderr is shown to the user and execution continues. |

For finer control, a hook can print JSON to stdout instead, for example to make a `PreToolUse` hook allow, deny, or ask for confirmation with a reason.

### Example: block edits to protected files

`.claude/hooks/protect-secrets.js`, wired up by the configuration above:

```javascript
const input = JSON.parse(require('fs').readFileSync(0, 'utf8'));
const path = input.tool_input?.file_path ?? '';

if (/(^|[\\/])\.env(\.|$)|appsettings\.Production\.json$/.test(path)) {
  console.error(`Blocked: ${path} is a protected file. Do not edit it.`);
  process.exit(2);
}
```

Claude receives the stderr message, so it knows why the edit was refused and can change approach.

### Risks and practices

- **Hooks run with your credentials**, with no confirmation prompt. Review any hook before adding it, and treat hooks in a cloned repository the way you would treat a build script.
- **Keep them fast.** Hooks run synchronously; a slow `PostToolUse` hook delays every edit.
- **Quote paths and validate input.** `tool_input` is model-generated; never pass it unquoted to a shell.
- **Check the syntax against your installed version.** Events and fields have been added over time.
- **Prefer hooks to instructions for hard rules**, and instructions to hooks for guidance that needs judgement.

## 4. Choosing between them

| Need | Use |
|---|---|
| Teach Claude a repeatable procedure or team standard | Skill |
| Keep a large or noisy task out of the main context, or run work in parallel | Subagent |
| Give a reviewer restricted tools or a different model | Subagent |
| Let Claude read or act on an external system | MCP server |
| Guarantee an action happens every time, or can never happen | Hook |

They compose: a subagent can be limited to specific MCP servers, a skill can describe how to use an MCP server's tools well, and a hook can enforce a rule on all of them.

## Related

- [AI-Assisted Development with VS Code, Claude Code & GitHub Copilot](../vscode/claude-code-copilot-guide.md) — sections 8 (Claude Skills), 9 (Claude Subagents), and 10 (Model Context Protocol)
