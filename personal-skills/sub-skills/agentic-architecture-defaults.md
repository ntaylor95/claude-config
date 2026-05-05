---
name: agentic-architecture-defaults
description: Use when designing or sketching an AI agent system, evaluating an agent architecture, or deciding between single-orchestrator and multi-agent approaches. Encodes Nicole's defaults inspired by the Shopify Sidekick patterns.
---

# Agentic Architecture Defaults

Nicole's defaults when sketching an agent system.

## Order of Decisions

1. **Start single-orchestrator + skills registry.** Don't reach for multi-agent unless the problem genuinely requires choreography.
2. **Budget tools.** Aim under ~20 tools across the whole agent. Quality over quantity.
3. **JIT-deliver instructions.** Don't preload everything into the system prompt; load context at the right moment.
4. **Structured output at parsing boundaries.** Schemas, XML tags, JSON-only modes where downstream code parses the result.
5. **Make the loop observable.** Log iterations, skill invocations, tool calls, and decision points. The loop itself is the artifact to debug.

## When Multi-Agent Is Actually Justified

- Truly parallelizable subtasks with independent contexts.
- Specialized models needed per subtask (cost/capability separation).
- Human-in-the-loop boundaries between agents (handoff with review).

If none of these apply, single-orchestrator wins.

## Anti-Patterns

- ❌ Bloated system prompt with every possible instruction loaded upfront.
- ❌ Tool sprawl — 50+ tools because each "felt useful."
- ❌ Multi-agent because it's trendy.
- ❌ Unstructured natural-language outputs at integration boundaries.
- ❌ Loops without iteration counters or telemetry.

## Reference Pattern

- **Control center / orchestrator** — runs the loop, manages conversation state, picks skills.
- **Skills registry** — modular capabilities, each with its own JIT instructions.
- **Tools** — invoked by skills; thin wrappers over real systems.
- **Conversation context** — stateful, scoped, observable.
