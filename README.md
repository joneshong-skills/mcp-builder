[English](README.md) | [繁體中文](README.zh.md)

# mcp-builder

A Claude Code skill that guides the creation of high-quality MCP (Model Context Protocol) servers in Python or TypeScript.

## Description

This skill provides a complete workflow for building MCP servers that wrap APIs or services into discoverable, well-documented tools for LLM agents. It covers the full development lifecycle: API research and planning, implementation with best practices, code review, testing, and evaluation.

## What It Does

- Guides workflow-oriented tool design (not just API endpoint wrappers)
- Provides language-specific implementation guides for Python (FastMCP) and TypeScript (MCP SDK)
- Includes best practices for naming, response formats, pagination, transport, and security
- Offers ready-to-use scripts for MCP connection handling and server evaluation
- Covers tool annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`)
- Includes an evaluation framework to test MCP server quality with LLM agents

## Install

Copy the skill directory into your Claude Code skills folder:

```
cp -r mcp-builder ~/.claude/skills/
```

Skills placed in `~/.claude/skills/` are auto-discovered by Claude Code. No additional registration is needed.

## Usage

Invoke the skill by asking Claude Code to build an MCP server. Example prompts:

- "Build an MCP server for the GitHub API"
- "Create an MCP server that wraps Slack"
- "Help me build MCP tools for a database"
- "Make a Python MCP server for Jira"

Claude will walk through the research, implementation, review, and evaluation phases automatically.
