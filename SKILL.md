---
name: mcp-builder
description: >-
  This skill should be used when the user asks to "build an MCP server",
  "create an MCP server", "make an MCP tool", "建立 MCP server",
  "寫 MCP 伺服器", "MCP 開發", "build MCP tools", mentions MCP server
  development, or discusses creating Model Context Protocol servers
  in Python or TypeScript.
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

#### 2.1 Project Structure

**Python:**
```
{service}_mcp/
├── server.py          # FastMCP instance + tool definitions
├── client.py          # API client with auth + error handling
├── models.py          # Pydantic models
├── formatters.py      # Response formatting
└── pyproject.toml
```

**TypeScript:**
```
{service}-mcp-server/
├── src/
│   ├── index.ts       # McpServer instance + tool registration
│   ├── client.ts      # API client
│   ├── schemas/       # Zod schemas
│   ├── tools/         # Tool implementations
│   └── types.ts
├── package.json
└── tsconfig.json
```

#### 2.2 Core Implementation Steps

1. **Set up API client** — Centralized auth, base URL, error handling
2. **Define input validation** — Pydantic models (Python) or Zod schemas (TypeScript)
3. **Implement tools** — One function per tool, with proper annotations
4. **Format responses** — Markdown for humans, JSON for structured data
5. **Handle pagination** — `limit`/`offset` params, return `has_more` + `next_offset`
6. **Handle errors** — Specific messages per HTTP status code (404, 403, 429)
7. **Set character limits** — Truncate long responses (~25,000 chars) with helpful messages

#### 2.3 Tool Implementation Pattern

**Python (FastMCP):**
```python
@mcp.tool(
    name="service_search_items",
    annotations={"readOnlyHint": True, "openWorldHint": True}
)
async def search_items(query: str, limit: int = 20) -> str:
    """Search for items matching the query.

    Args:
        query: Search query string
        limit: Maximum results to return (1-100, default 20)
    """
    results = await api_client.search(query, limit=limit)
    return format_results_as_markdown(results)
```

**TypeScript (MCP SDK):**
```typescript
server.registerTool(
  "service_search_items",
  {
    title: "Search Items",
    description: "Search for items matching the query",
    inputSchema: {
      query: z.string(),
      limit: z.number().min(1).max(100).default(20),
    },
    annotations: { readOnlyHint: true, openWorldHint: true },
  },
  async ({ query, limit }) => {
    const results = await apiClient.search(query, limit);
    return { content: [{ type: "text", text: formatResults(results) }] };
  }
);
```

#### 2.4 Response Format Guidelines

- Use **Markdown** for agent-readable responses (headers, tables, lists)
- Include human-readable identifiers (names alongside IDs)
- Use readable timestamps (not raw Unix timestamps)
- For large datasets, paginate and indicate remaining items

#### 2.5 Best Practices

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
python -c "import py_compile; py_compile.compile('server.py', doraise=True)"
python server.py --help
```

**TypeScript:**
```bash
npm run build
npx ts-node src/index.ts --help
```

#### 3.3 Integration Test

Configure the server in Claude Code's MCP settings and verify tools work:
```bash
# Add to ~/.claude/settings.json under mcpServers
claude mcp add <server-name> -- <command> <args>
```

### Phase 4: Create Evaluations (Optional)

Create 10 evaluation questions to measure server quality with LLM agents.

1. Run `python3 ~/.claude/skills/mcp-builder/scripts/evaluation.py` against the server
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
// ~/.claude/settings.json
{
  "mcpServers": {
    "my-server": {
      "command": "python3",
      "args": ["/path/to/server.py"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

Or via CLI:
```bash
claude mcp add my-server -- python3 /path/to/server.py
```

## Continuous Improvement

This skill evolves with each use. After every invocation:

1. **Reflect** — Identify what worked, what caused friction, and any unexpected issues
2. **Record** — Append a concise lesson to `lessons.md` in this skill's directory
3. **Refine** — When a pattern recurs (2+ times), update SKILL.md directly

### lessons.md Entry Format

```
### YYYY-MM-DD — Brief title
- **Friction**: What went wrong or was suboptimal
- **Fix**: How it was resolved
- **Rule**: Generalizable takeaway for future invocations
```

Accumulated lessons signal when to run `/skill-optimizer` for a deeper structural review.

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
