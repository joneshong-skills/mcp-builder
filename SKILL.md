---
name: mcp-builder
description: "mcp, builder, build, server, create, tool, tools, 建立 MCP server, 寫 MCP 伺服器, MCP 開發"
version: 0.2.0
tools: Read, Glob, Grep, Bash, Edit, Write
argument-hint: "<service name or API to wrap>"
---

# MCP Server Builder

Guide the creation of high-quality MCP (Model Context Protocol) servers that provide
tools for LLM agents. An MCP server wraps APIs or services into discoverable,
well-documented tools that agents can use effectively.

## Workflow

### Phase 1: Deep Research and Planning

Before writing any code, understand the target API and plan the tool design.

#### 1.1 Understand Modern MCP Design

MCP servers should provide **workflow-oriented tools**, not just API endpoint wrappers.

| Approach | Description | Example |
|----------|-------------|---------|
| API Coverage | One tool per endpoint | `get_user`, `list_users`, `update_user` |
| Workflow Tools | Tools matching natural tasks | `find_user_activity`, `manage_user_permissions` |

Prefer workflow tools. Key principles:
- **Tool naming**: Use `{service}_{action}` format (e.g., `github_search_issues`)
- **Discoverability**: Tool names and descriptions must be self-explanatory to agents
- **Context efficiency**: Return only what agents need, formatted for readability
- **Annotations**: Mark tools as `readOnlyHint`, `destructiveHint`, etc.

#### 1.2 Apply Tool Design Principles

Before defining tools, read `references/tool-design.md` for critical design guidance:

- **Consolidation**: If two tools could handle the same input, merge them. Fewer tools = higher agent success rate.
- **Architectural reduction**: Favor general-purpose primitives over many specialized wrappers.
  A real case study achieved 100% success (from 80%) by reducing 17 tools to 2.
- **Description engineering**: Every tool description must answer: what it does, when to use it
  (vs similar tools), what inputs it needs, and what it returns.
- **Target 5-15 tools** per server. Below 5 may lack coverage; above 25 degrades agent selection.

#### 1.3 Study the API

1. Read the target API documentation thoroughly
2. Identify the most valuable operations for agent workflows
3. Plan 5-15 focused tools (not 50+ thin wrappers)
4. Determine authentication method (API key, OAuth 2.1)

#### 1.4 Choose Language

| Factor | Python (FastMCP) | TypeScript (MCP SDK) |
|--------|-------------------|----------------------|
| Best for | Rapid prototyping, data/ML APIs | Production services, type safety |
| Validation | Pydantic BaseModel | Zod schemas |
| Server name | `{service}_mcp` | `{service}-mcp-server` |
| Reference | `references/python_mcp_server.md` | `references/node_mcp_server.md` |

### Phase 2: Implementation

Project layout (node), client/validation/tool scaffolding and full worked examples live in
`references/python_mcp_server.md` (FastMCP) and `references/node_mcp_server.md` (TS SDK) — follow them.
Python layout: `{service}_mcp/` → `server.py` (FastMCP instance + tools), `client.py` (API client, auth + errors), `models.py` (Pydantic), `formatters.py` (response formatting), `pyproject.toml`.
Contract every tool must meet regardless of language:

- Responses agent-readable: Markdown, names alongside IDs, readable timestamps (not raw Unix)
- Pagination via `limit`/`offset`, returning `has_more` + `next_offset`
- Long responses truncated at ~25,000 chars with a message saying how to get the rest
- Errors mapped per HTTP status (see Common Error Handling below)

#### 2.1 Best Practices

Read `references/mcp_best_practices.md` for comprehensive guidelines on:
- Transport selection (stdio vs streamable HTTP)
- Security (OAuth 2.1, input validation, DNS rebinding protection)
- Error handling patterns
- Tool annotation semantics

### Phase 3: Review and Test

#### 3.1 Code Quality Checks

- [ ] No duplicated logic — extract shared utilities
- [ ] All tools have descriptive names with service prefix
- [ ] Error messages are clear and actionable
- [ ] Pagination implemented where applicable
- [ ] Character limits prevent context overflow

#### 3.2 Build and Test

**Python:**
```bash
~/.local/bin/python3 -c "import py_compile; py_compile.compile('server.py', doraise=True)"
~/.local/bin/python3 server.py --help
```

**TypeScript:**
```bash
npm run build
npx ts-node src/index.ts --help
```

#### 3.3 Integration Test

Register the server with Claude Code and verify tools work:
```bash
# -s user writes to ~/.claude.json top-level mcpServers (default scope is local → projects.<cwd>.mcpServers;
# ~/.claude/settings.json has no mcpServers key)
claude mcp add -s user <server-name> -- <command> <args>
```
On this machine the main line only mounts `mcpproxy` (sole entry in `~/.claude.json`); a server meant for
daily use is added as an mcpproxy upstream, and direct `claude mcp add` is for isolated testing only.

### Phase 4: Create Evaluations (Optional)

Create 10 evaluation questions to measure server quality with LLM agents.

1. Run `~/.local/bin/python3 ~/.claude/skills/mcp-builder/scripts/evaluation.py` against the server
2. Questions must be independent, read-only, realistic, and verifiable
3. Output format: XML with `<qa_pair>` elements

See `references/evaluation.md` for detailed guidelines and examples.

## Quick Reference

### Tool Annotation Cheat Sheet

| Annotation | Meaning | Example |
|------------|---------|---------|
| `readOnlyHint: true` | No side effects | Search, list, get |
| `destructiveHint: true` | May delete/overwrite | Delete, update |
| `idempotentHint: true` | Safe to retry | Update (PUT), delete |
| `openWorldHint: true` | External interaction | API calls, web requests |

### Common Error Handling

| Status | Action |
|--------|--------|
| 400 | Return validation error details |
| 401/403 | "Authentication failed — check API key" |
| 404 | "Resource not found: {id}" |
| 429 | "Rate limited — retry after {seconds}s" |
| 500+ | "Server error — try again later" |

### MCP Config for Claude Code (macOS)

```json
// ~/.claude.json  (user-scope; ~/.claude/settings.json does not carry mcpServers)
// command must be an absolute path here: `claude mcp add` below expands ~ for you,
// and ${VAR} expansion is documented only for project-scope .mcp.json
{
  "mcpServers": {
    "my-server": {
      "command": "/absolute/path/to/python3",
      "args": ["/path/to/server.py"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

Or via CLI:
```bash
claude mcp add -s user my-server -- ~/.local/bin/python3 /path/to/server.py
```

## Additional Resources

### Reference Files
- **`references/tool-design.md`** — Tool design principles: consolidation, architectural reduction, description engineering, anti-patterns
- **`references/mcp_best_practices.md`** — Universal MCP guidelines: naming, responses, pagination, transport, security
- **`references/python_mcp_server.md`** — Python/FastMCP implementation guide with complete examples and quality checklist
- **`references/node_mcp_server.md`** — TypeScript/Node.js implementation guide with Zod schemas and quality checklist
- **`references/evaluation.md`** — Evaluation question design: requirements, good/bad examples, verification process

### Scripts
- **`scripts/connections.py`** — Lightweight MCP connection handling (stdio, SSE, HTTP)
- **`scripts/evaluation.py`** — Evaluation harness for testing MCP servers with Claude
- **`scripts/example_evaluation.xml`** — Example evaluation file format
