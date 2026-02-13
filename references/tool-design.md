# Tool Design — Principles for Agent-Effective Tools

How to design MCP tools that agents can actually use well. Based on architectural
reduction patterns and description engineering research.

---

## Table of Contents

- [The Consolidation Principle](#the-consolidation-principle)
- [Architectural Reduction](#architectural-reduction)
- [Description Engineering](#description-engineering)
- [Anti-Patterns](#anti-patterns)
- [Tool Collection Management](#tool-collection-management)

---

## The Consolidation Principle

> If a human engineer cannot definitively say which tool should be used in a given
> situation, an agent cannot be expected to do better.

When two tools have overlapping use cases, the agent will frequently pick the wrong one.
**Favor fewer, comprehensive tools over many specialized ones.**

### Decision Framework

| Signal | Action |
|--------|--------|
| Two tools could both handle the same input | Merge into one with a `mode` parameter |
| A tool is only used in combination with another | Combine into a single workflow tool |
| Users describe a task that spans 3+ tools | Create a composite tool for that workflow |
| A tool has < 5% usage in production | Remove or merge into a broader tool |

### Before/After Example

**Before** (5 tools — agent often picks wrong one):
```
search_users_by_name
search_users_by_email
search_users_by_id
search_users_by_role
list_all_users
```

**After** (1 tool — unambiguous):
```
find_users(query, filter_by="auto")
# Accepts name, email, ID, or role — auto-detects query type
# filter_by can override: "name", "email", "id", "role", or "all" to list
```

---

## Architectural Reduction

### Case Study: Text-to-SQL Agent — 17 Tools → 2

A production text-to-SQL agent originally had 17 specialized tools:
`ListTables`, `GetTableSchema`, `ValidateQuery`, `ExecuteQuery`, `FormatResults`,
`GetTableRelationships`, `SuggestJoins`, `OptimizeQuery`, etc.

**Replaced with 2 general-purpose primitives:**

| Tool | Purpose |
|------|---------|
| `ExecuteCommand` | Run any shell command (inspect schemas, validate, format) |
| `ExecuteSQL` | Run any SQL query (all data operations) |

**Results:**

| Metric | 17 Tools | 2 Tools | Change |
|--------|----------|---------|--------|
| Success rate | 80% | 100% | +20% |
| Avg steps per task | 12 | 7 | -42% |
| Avg tokens per task | 15,000 | 9,500 | -37% |
| Execution time | 45s | 13s | -71% |

**Why it worked:**
- Agent spent fewer tokens on tool selection reasoning
- No more "wrong tool" errors requiring backtracking
- General primitives let the agent compose its own workflows
- The agent's reasoning ability replaced the rigid tool pipeline

### When to Apply Architectural Reduction

| Condition | Recommendation |
|-----------|---------------|
| Agent frequently chains 3+ tools for one task | Merge the chain into one tool |
| Agent backtracks due to wrong tool selection | Reduce tool count or add `mode` param |
| Multiple tools wrap the same underlying API | Expose the API more directly |
| Tool descriptions overlap significantly | Consolidate with parameters |
| Success rate < 85% with current tool set | Profile which tools cause failures |

### When NOT to Reduce

- Tools with genuinely different side effects (read vs write vs delete)
- Tools accessing different external services
- Tools requiring different authentication scopes
- When tool annotations (readOnly, destructive) would become ambiguous

---

## Description Engineering

Every tool description must answer four questions:

1. **What does it do?** — One clear sentence
2. **When should it be used?** — Differentiation from similar tools
3. **What inputs does it need?** — Parameters with types and constraints
4. **What does it return?** — Output format and content

### Weak vs Strong Descriptions

**Weak:**
```
name: "get_data"
description: "Gets data from the system"
```
Problems: What data? Which system? When to use vs other tools?

**Strong:**
```
name: "github_search_issues"
description: "Search GitHub issues and pull requests by keyword, label, or
  assignee. Returns titles, URLs, status, and assignees. Use this instead
  of github_list_repo_issues when you need to filter by search query.
  Returns max 30 results; use 'page' for pagination."
```

### Description Quality Checklist

| Criterion | Check |
|-----------|-------|
| **Clarity** | Can a human immediately understand what this tool does? |
| **Completeness** | Are all parameters, defaults, and constraints documented? |
| **Differentiation** | Does it explain when to use THIS tool vs similar ones? |
| **Output format** | Does it describe what the response looks like? |
| **Limits** | Are rate limits, max results, and truncation documented? |

### Parameter Engineering

| Practice | Bad | Good |
|----------|-----|------|
| Name | `q`, `t`, `f` | `query`, `time_range`, `format` |
| Type info | `id: any` | `issue_number: int (1-99999)` |
| Defaults | (none documented) | `limit: int = 20 (max 100)` |
| Enum values | `type: string` | `type: "bug" \| "feature" \| "task"` |
| Required vs optional | (ambiguous) | Required params first, optional with defaults |

---

## Anti-Patterns

### 1. The API Mirror

Exposing every API endpoint as a separate tool.

```
# Bad: 47 tools, one per REST endpoint
get_user, list_users, create_user, update_user, delete_user,
get_user_settings, update_user_settings, get_user_avatar, ...
```

Fix: Group into 5-10 workflow-oriented tools.

### 2. The Ambiguous Pair

Two tools with overlapping functionality and unclear boundaries.

```
# Bad: When should agent use which?
search_items(query)
find_items(filter)
```

Fix: Merge into one tool or make descriptions explicitly differentiate.

### 3. The Silent Failure

Tool returns empty/null without explanation.

```
# Bad
return ""

# Good
return "No results found for query 'xyz'. Try broader search terms or check spelling."
```

### 4. The Token Bomb

Tool returns massive unfiltered responses.

```
# Bad: Returns all 10,000 rows
return json.dumps(all_records)

# Good: Paginated with summary
return f"Showing {len(page)} of {total} results.\n{format_page(page)}\n\nUse offset={next_offset} for more."
```

### 5. The Black Box Error

```
# Bad
return "Error: something went wrong"

# Good
return "Error 429: Rate limited by GitHub API. Retry after 60 seconds. Consider using a smaller page_size to reduce API calls."
```

---

## Tool Collection Management

### Size Guidelines

| Tool Count | Risk | Action |
|------------|------|--------|
| 1-5 | May be too few — check coverage | Ensure core workflows are covered |
| 5-15 | Optimal range | Maintain clear differentiation |
| 15-25 | Getting large — review usage | Profile and merge low-usage tools |
| 25+ | Agent selection degrades | Apply architectural reduction |

### Namespace Strategy

When an MCP server coexists with others, prefix all tools:

```
# Without prefix — collision risk
search, create, delete

# With prefix — unambiguous
jira_search, jira_create, jira_delete
```

### Measuring Tool Effectiveness

Track these metrics in production:

| Metric | Target | Red Flag |
|--------|--------|----------|
| Tool selection accuracy | > 95% | Agent picks wrong tool > 5% of the time |
| Backtrack rate | < 10% | Agent frequently retries with different tool |
| Avg tools per task | < 5 | Complex chains indicate missing composite tools |
| Unused tools | 0 | Remove tools with 0 usage over 30 days |
