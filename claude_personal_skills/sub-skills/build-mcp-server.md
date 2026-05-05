---
name: build-mcp-server
description: Use when Nicole is building, scaffolding, or designing a personal or project MCP server. Covers prompt/resource/tool decisions, FastMCP-friendly layout, thin-server architecture, naming conventions, and schema-resource patterns without employer-specific details.
---

# Build MCP Server

Nicole's defaults for building MCP servers.

## Service Type Decision

| Use a... | When... |
|---|---|
| Prompt | Reusable instruction template or workflow. |
| Resource | Static or semi-static context: schema, config, vocabulary, examples. |
| Tool | Performs an action, fetches dynamic data, or has side effects. |

## Architecture Defaults

- Keep the server thin. Expose the right prompts/resources/tools and delegate execution to systems that already do the work well.
- Prefer resources for stable schema/context and tools for live actions.
- Keep the tool surface small and obvious.
- Design prompts so the model asks for missing scope before acting.
- Make failures legible: clear errors, typed outputs where possible, and enough logging to debug.

## Naming & Layout

- Directory: kebab-case, for example `personal-notes-mcp`.
- Python module: snake_case, for example `personal_notes_mcp.py`.
- Constants: `constants.py` for `SERVER_NAME`, `SERVER_VERSION`, and shared resource URIs.
- README at the root with setup, run, test, and usage examples.
- Prefer `uv` and `pyproject.toml` for Python projects unless the project already uses a different standard.

## Prompt Pattern

Every prompt should:

1. Take the user's question or task as input.
2. Ask for clarification if scope, dates, data shape, or destination is missing.
3. Reference relevant resources by URI instead of embedding bulky context.
4. State required constraints or filters explicitly.
5. Include a concrete output example when reliability matters.

## Resource Pattern

- One resource per logical entity, for example `schema://customers` or `notes://writing-style`.
- Include names, fields, types, constraints, and gotchas.
- Include one small example query or usage pattern.
- Avoid private data in reusable resources; use placeholders or synthetic examples.

## Tool Pattern

A tool should have:

- A narrow verb-first name.
- Typed arguments.
- Clear side-effect expectations.
- Predictable return shape.
- Human-safe error messages.

## Anti-Patterns

- Reimplementing a database, file system, or API client when another MCP/tool already does it well.
- Prompts that silently assume a date range or scope.
- Schema resources without examples.
- Tools with broad vague names like `do_task` or `handle_request`.
