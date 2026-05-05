---
name: nicole-workflow
description: Always-on personal context for how Nicole designs, builds, reviews, documents, and collaborates. Load to align Claude with her engineering principles, code hygiene, documentation preferences, and AI-assisted workflow defaults.
---

# Nicole — Workflow Context

Use this to set defaults. Load a sub-skill when a task maps cleanly to one.

## Engineering Principles

- Ownership of code belongs to the human shipping it. "The AI suggested it" is not a defense.
- AI raises the bar; it does not lower standards for readability, maintainability, correctness, or user value.
- Skills and AI instruction files should live close to the work they support.
- Prefer a single agentic loop plus targeted skills/tools before reaching for multi-agent orchestration.
- Use just-in-time instructions, not bloated system prompts.
- Tool budgets matter. Prefer fewer, higher-quality tools.
- Use structured output where reliability matters: schemas, XML-tagged sections, or JSON-only boundaries.
- Thin MCP servers over fat ones. Delegate execution to systems that already do it well.
- If a manual task repeats, look for the reusable workflow.

## Code & Review Hygiene

- Verify AI-suggested libraries, APIs, and dependencies before adopting.
- Inspect commits before pushing.
- For risky changes: use feature flags, staged rollout, and observable checkpoints.
- Validate observable behavior, not implementation details.
- Keep pull requests small, coherent, and human-readable.
- Review consistently rather than in large irregular batches.

## Documentation Defaults

- READMEs should explain why the code exists, how to run/test/deploy/observe it, and what a new contributor should know.
- Project-specific `CLAUDE.md` or skill files should be committed with the code they support.
- Capture decisions and rationale when work affects more than one person or system.

## On-Demand Skills

- `build-mcp-server` — building or designing an MCP server.
- `write-readmes` — generating or reviewing README content.
- `agentic-architecture-defaults` — designing or reviewing an agentic system.
- `frame-for-reviewers` — drafting review comments, pushback, or technical replies.
- `annual-review-voice` — career, self-review, or growth-plan writing.
- `conference-mentorship-voice` — approachable talks, bios, mentoring, or AI-demystifying content.
- `clarify-before-building` — ambiguous scope or missing inputs.
