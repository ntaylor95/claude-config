# Nicole's Personal Skills Library

Vendor-neutral, portable skill files describing how Nicole works, what she knows, and how she wants AI to interact with her. Designed to be loaded by an LLM directly or registered with a personal MCP server.

## Structure

```
nicole-skills/
├── README.md                        ← this file
├── domain.md                        ← always-on: SME on Nicole's domain
├── workflow.md                      ← always-on: SME on how Nicole works
├── behaviors.md                     ← always-on: SME on how to interact with Nicole
└── sub-skills/                      ← triggered by description, loaded on demand
    ├── build-mcp-server.md
    ├── write-readmes.md
    ├── agentic-architecture-defaults.md
    ├── frame-for-reviewers.md
    ├── annual-review-voice.md
    ├── conference-mentorship-voice.md
    ├── clarify-before-building.md
    ├── parent-logistics-mode.md
    └── hackathon-mode.md
```

## How to Use

- **Top-level files** (`domain.md`, `workflow.md`, `behaviors.md`) load every conversation. They establish always-on context.
- **Sub-skills** load only when their `description` field matches the task at hand. The top-level files reference them by name.

## Format

Every file follows the standard `SKILL.md` convention:

```yaml
---
name: skill-name
description: When to load this skill / what it covers
---

# Title
[content]
```

The `description` field is what tells the LLM whether to load the skill. Keep descriptions specific so triggering is reliable.

## Maintenance

- Edit freely. This is a living document.
- Add new sub-skills under `sub-skills/` and reference them in the relevant top-level file.
- Keep vendor-neutral. No proprietary schemas, no internal project names, no teammate-identifying details.
- If a top-level file grows past ~300 lines, consider splitting it.
