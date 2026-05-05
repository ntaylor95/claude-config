---
name: agentic-architecture-defaults
description: Use when designing, reviewing, or troubleshooting an AI agent system. Applies Nicole's defaults: single orchestrator before multi-agent, just-in-time skill loading, small tool budgets, structured outputs at integration boundaries, and observable agent loops.
---

# Agentic Architecture Defaults

Nicole's defaults when sketching or reviewing an agentic system.

## Order of Decisions

1. Start with a single orchestrator plus a skills registry. Do not use multi-agent architecture unless the problem genuinely requires choreography.
2. Budget tools. Prefer fewer, higher-quality tools over a broad grab bag.
3. Load instructions just in time. Avoid preloading every possible rule into the system prompt.
4. Use structured output at parsing boundaries: schemas, XML tags, or strict JSON where downstream code depends on it.
5. Make the loop observable. Log iterations, skill selections, tool calls, decision points, retries, and failure modes.

## When Multi-Agent Is Justified

- Subtasks are truly parallel and have independent context.
- Different subtasks need different model capabilities or cost profiles.
- Human review gates naturally split the workflow.
- A single context window cannot reasonably carry the workflow state.

If none of these apply, single-orchestrator usually wins.

## Reference Pattern

- Orchestrator: manages loop state, selects skills, chooses tools, and decides when to stop.
- Skills registry: modular, just-in-time instructions.
- MCP tools: thin interfaces to real systems.
- MCP resources: stable context, schemas, vocabulary, examples.
- Prompts: reusable workflows or task templates.
- Observability: structured traces of the loop.

## Anti-Patterns

- Bloated system prompt with every instruction always loaded.
- Tool sprawl because each tool "might be useful."
- Multi-agent because it sounds advanced.
- Natural-language blobs at integration boundaries.
- Agent loops without counters, traces, or stop conditions.
