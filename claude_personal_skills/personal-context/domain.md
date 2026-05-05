---
name: nicole-domain
description: Always-on personal context for Nicole's durable domain knowledge: software engineering, frontend and full-stack systems, AI-assisted development, MCP servers, data/SQL, logistics/e-commerce familiarity, and preferred tools. Keep company-neutral.
---

# Nicole — Domain Context

Use this to calibrate technical depth and vocabulary. Do not over-explain concepts Nicole already knows.

## Software Engineering

- 20+ years of software engineering experience.
- Frontend primary; comfortable in TypeScript and React.
- Full-stack reviewer; intentionally growing backend depth through hands-on builds.
- Values maintainability, observable behavior, small coherent changes, and documentation that helps the next person.

## AI-Assisted Development

- Uses Claude Code, Claude.ai, Claude Platform, and personal MCP servers.
- Builds skills to encode repeatable workflows rather than relying on one-off prompting.
- Prefers just-in-time instructions over massive always-on prompts.
- Thinks of skills, resources, and tools as separate responsibilities.

## MCP / Agentic Systems

- Prefers thin MCP servers that expose prompts/resources/tools and delegate execution to the right underlying system.
- Defaults to a single orchestrator plus a skills registry unless the problem genuinely requires multi-agent choreography.
- Cares about observability of agent loops: iterations, tool calls, skill selection, decision points, and failure modes.

## Data / SQL

- Comfortable writing SQL and designing schema guidance for LLMs.
- Cares about required filters, tenant boundaries, soft-delete flags, partition pruning, and example queries.
- Default to the user's stated dialect. If unspecified, ask or use a generic SQL style and state the assumption.

## Business / Product Domains

- Familiar with e-commerce, shipping/logistics, order flow, fulfillment, returns, and seller-facing product workflows.
- Treat this as domain fluency, not as company-specific knowledge.
- Do not include proprietary system names, customer names, internal schemas, or employer-specific implementation details.

## Preferred Tooling Patterns

- Python for backend, scripting, MCP servers, and data workflows.
- `uv` + `pyproject.toml` for Python environment management where possible.
- TypeScript + React for frontend work.
- Generic CI/CD, observability, issue tracking, and analytics systems should be referenced by role unless the user gives exact tools for the current project.
