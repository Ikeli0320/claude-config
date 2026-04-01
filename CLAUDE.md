# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a personal configuration and documentation workspace for Claude CLI / Claude Code / Claude Cowork tooling. It tracks plugins, MCP servers, skills, and related configurations so they can be synchronized across multiple development machines.

This repository does **not** contain application source code — there is no build system, test suite, or linting pipeline.

## What Lives Here

- Documentation and notes about installed Claude Code skills, plugins, and MCP servers
- Configuration references and setup instructions for replicating the Claude Code environment on new machines
- Any helper scripts or templates created for cross-machine portability

## Key External Paths (Windows)

| Path | Description |
|------|-------------|
| `~/.claude/settings.json` | Global Claude Code settings (permissions, update channel) |
| `~/.claude/skills/` | Installed skills (SKILL.md per skill) |
| `~/.claude/plugins/` | Installed plugins and marketplace cache |
| `~/.claude/projects/` | Per-project session data and memory |

## Conventions

- Language: The user communicates in Traditional Chinese (繁體中文); respond in the same language unless the context is code or English documentation.
- This workspace is on Windows 11 using Git Bash — use Unix-style paths and shell syntax.
- When documenting configurations, always note which scope they belong to (global `settings.json` vs project-level `settings.local.json`).
