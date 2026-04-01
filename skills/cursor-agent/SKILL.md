---
name: cursor-agent
description: This skill should be used when you need to delegate implementation tasks to Cursor Agent as a headless subagent. Cursor Agent can autonomously read, write, and modify files in the project using AI. Use this skill to run parallel or sequential coding tasks without consuming Claude Code's context window.
---

# Cursor Agent Subagent

Cursor Agent is a standalone CLI that runs AI coding tasks headlessly. Use it to delegate file writing, refactoring, or implementation work.

## Binary Location (Windows)

```
%LOCALAPPDATA%\cursor-agent\agent.ps1
# Typical path: C:\Users\<USERNAME>\AppData\Local\cursor-agent\agent.ps1
```

## Core Invocation Pattern

Always invoke via `pwsh` to ensure PowerShell execution. Use `--trust` + `--yolo` for fully automated runs. Capture output to a file since long commands get backgrounded by Claude Code's Bash tool.

```bash
pwsh -Command "
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
cd 'C:\path\to\your\project'
\$output = & \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" \`
  '-p' 'YOUR PROMPT HERE' \`
  '--trust' \`
  '--yolo' \`
  '--output-format' 'text' 2>&1
\$output | Out-File 'cursor-output.txt' -Encoding UTF8
"
```

Then read result:
```bash
cat cursor-output.txt
```

## Key Flags

| Flag | Description |
|------|-------------|
| `-p "prompt"` | Headless / non-interactive mode with initial prompt |
| `--trust` | Trust workspace (required first run or new dirs) |
| `--yolo` / `-f` | Auto-approve all file writes and shell commands |
| `--output-format text` | Clean text output (vs json / stream-json) |
| `--output-format json` | Structured output for parsing |
| `--model claude-sonnet-4` | Override model (default: auto) |
| `--plan` | Read-only planning mode (no writes) |
| `--mode ask` | Q&A / explanation mode (no writes) |
| `--continue` | Continue previous session |
| `--resume <chatId>` | Resume specific session |
| `--list-models` | List available models |

## Output Formats

- `text` — Clean final answer, best for reading results
- `json` — Structured with tool calls, good for parsing
- `stream-json` — Real-time streaming events

## Workflow Patterns

### 1. Implement a Feature (writes files)
```bash
pwsh -Command "
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
cd 'C:\path\to\your\project'
& \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" \`
  '-p' 'Create a Next.js server component that fetches data from Supabase. Use Tailwind CSS.' \`
  '--trust' '--yolo' '--output-format' 'text' | Out-File 'cursor-output.txt' -Encoding UTF8
"
cat cursor-output.txt
```

### 2. Planning Only (no writes)
```bash
pwsh -Command "
cd 'C:\path\to\your\project'
& \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" \`
  '-p' 'Analyze the current codebase and suggest an implementation plan.' \`
  '--trust' '--plan' '--output-format' 'text' | Out-File 'cursor-plan.txt' -Encoding UTF8
"
```

### 3. Short Inline Test
```bash
pwsh -Command "
& \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" '-p' 'say hello' '--trust' 2>&1
"
```

## Collaboration Model: Claude Code + Cursor Agent

Use Claude Code (me) as the **orchestrator**:
- Designs architecture and task breakdown
- Assigns tasks to Cursor Agent
- Reviews output and runs validation (type-check, build)
- Handles git operations (branch, commit, PR)

Use Cursor Agent as the **implementer**:
- Writes boilerplate and repetitive code
- Implements well-defined modules with clear specs
- Can handle multiple files per task

## Best Practices

1. **Give Cursor Agent precise context** — include file paths, function names, types to use
2. **One module per invocation** — don't ask it to implement the entire system at once
3. **Always validate after** — run `npm run type-check` and `npm run build` after cursor-agent writes
4. **Use `--plan` first** for complex tasks — review the plan before running with `--yolo`
5. **Output to file** — long commands get backgrounded; write to file and read separately

## Version Check

```bash
pwsh -Command "& \"\$env:LOCALAPPDATA\cursor-agent\agent.ps1\" '--version'"
```
