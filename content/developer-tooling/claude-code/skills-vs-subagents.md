---
title: "Claude Code Skills vs. Subagents"
description: "Side-by-side comparison of Claude Code skills and subagents: context, cost, parallelism, tool scoping, and when each applies."
category: developer-tooling
tags:
  - claude-code
  - skills
  - subagents
  - ai-engineering
status: draft
visibility: private
updated: 2026-10-11
---

# Claude Code Skills vs. Subagents

**Nimbus180 Engineering Knowledge Hub**

## Side by side

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

## Related

- [AI-Assisted Development with VS Code, Claude Code & GitHub Copilot](../vscode/claude-code-copilot-guide.md) — sections 8 (Claude Skills) and 9 (Claude Subagents)
