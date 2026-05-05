---
name: nicole-workflow
description: Always-on SME on how Nicole works — engineering principles, system design defaults, code and review hygiene, and writing voice for technical documents. Loaded every session to align approach with her established working patterns.
---

# Nicole — Workflow SME

Always-on context describing how Nicole prefers to design, build, review, and document. Use this to set defaults; load specific sub-skills when triggered.

## Engineering Principles

- **Ownership of code belongs to the human shipping it.** "The AI suggested it" is not a defense.
- **AI raises the bar; it doesn't lower it.** Standards on readability, maintainability, and user value remain.
- **Skills live with the features they serve.** AI instruction files belong in the repo next to the code they configure.
- **Agentic loop > multi-agent.** Don't reach for choreography when a single orchestrator + good tools will do.
- **JIT instructions, not bloated system prompts.** Right context at the right moment.
- **Tool budgets matter.** Aim under ~20 tools per agent.
- **Structured output where reliability matters.** Schemas and XML-tagged sections, not vibes.
- **Thin MCP servers over fat ones.** Delegate execution to tools that already do it well.
- **Automation mindset.** If doing it manually twice, look for the way to stop.

## Code & Review Hygiene

- Verify any AI-suggested library or dependency before adopting.
- Check commits before pushing.
- For changes that touch the guts of a product: feature flags + gradual rollout.
- Validate observable behavior, not implementation details.
- Keep PRs small, coherent, human-readable.
- Review PRs consistently — not in feast/famine waves.

## Documentation Defaults

- READMEs follow Nicole's contract (see `write-readmes` sub-skill).
- Skills/CLAUDE.md files committed alongside features.
- Decisions and rationale documented for cross-team initiatives.

## Available Sub-Skills

Load these when their description matches the situation:

- `build-mcp-server` — building or scaffolding an MCP server
- `write-readmes` — generating or reviewing a README
- `agentic-architecture-defaults` — designing an agent system
- `frame-for-reviewers` — drafting PR comments, review feedback, technical writing
- `annual-review-voice` — performance review / self-evaluation content
- `conference-mentorship-voice` — internal talks, mentorship, public-facing dev advocacy
- `clarify-before-building` — ambiguous-scope task; need to pin down inputs before starting
