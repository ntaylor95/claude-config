---
name: build-mcp-server
description: Use when Nicole is building, scaffolding, or designing an MCP server. Covers FastMCP patterns, the prompt vs resource vs tool decision, naming conventions, and her preference for thin servers that delegate execution to other MCP servers (like Toolbox for Databases).
---

# Build MCP Server

Nicole's defaults for building MCP servers.

## Service Type Decision

| Use a... | When... |
|---|---|
| **Prompt** | Reusable instruction template or workflow. LLM picks it based on user intent. |
| **Resource** | Static or semi-static context (schema, config, vocabulary). Loaded as background. |
| **Tool** | Performs an action with side effects, or fetches dynamic data. LLM decides when to call. |

## Architecture Defaults

- **Thin server, fat ecosystem.** If Toolbox for Databases (or another MCP server) already executes well, delegate to it. Don't reimplement DB access.
- Provide prompts and schema resources that *guide* downstream execution.
- Keep tool count under ~20 across the whole agent.

## Naming & Layout

- **Directory:** kebab-case (e.g., `seller-reports-mcp`)
- **Python file:** snake_case (e.g., `seller_reports_mcp.py`)
- **Constants module:** `constants.py` for `SERVER_NAME`, `SERVER_VERSION`
- **README** at the root following Nicole's README contract

## Prompt Pattern

Every prompt should:
1. Take the user's question as input.
2. If date range / scope is missing, instruct the LLM to ask for clarification first.
3. Reference a schema resource so the LLM has the right column/table names.
4. Specify required filters (soft-delete flags, tenant joins, partition pruning).
5. Provide a concrete SQL or output example.

## Resource Pattern

- One resource per logical entity (e.g., `schema://orders`, `schema://shipments`).
- Include columns, types, soft-delete flags, JOIN keys, partition columns, and one example query.
- Include any "gotcha" filters that are non-obvious.

## Anti-Patterns

- ❌ Server reimplements DB access when Toolbox would do.
- ❌ Prompts assume a date range; they should always ask if missing.
- ❌ Schema resources without example queries.
- ❌ Prompts hallucinating field names — they should defer to the resource.
