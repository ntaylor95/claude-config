# Nicole's Personal Claude Skills

Company-neutral Claude Skills and personal context files for a personal MCP server.

These files capture durable preferences, workflows, and reusable operating modes without employer-specific names, proprietary schemas, internal project details, or teammate-identifying information.

## Suggested Layout

```text
claude-personal-skills/
├── README.md
├── personal-context/
│   ├── behaviors.md
│   ├── domain.md
│   └── workflow.md
└── sub-skills/
    ├── agentic-architecture-defaults.md
    ├── annual-review-voice.md
    ├── build-mcp-server.md
    ├── clarify-before-building.md
    ├── conference-mentorship-voice.md
    ├── frame-for-reviewers.md
    ├── hackathon-mode.md
    ├── parent-logistics-mode.md
    └── write-readmes.md
```

## How to Use With a Personal MCP Server

Use the three files in `personal-context/` as always-on resources or bootstrap context:

- `behaviors.md` — how Nicole wants the assistant to communicate.
- `domain.md` — durable domain and technical context.
- `workflow.md` — engineering, documentation, and AI-assisted work defaults.

Use files in `sub-skills/` as on-demand skills. Register their `description` metadata as the trigger text and load the body only when relevant.

## Privacy / Portability Rules

- Keep these files employer-neutral.
- Do not include proprietary schemas, internal system names, internal project names, customer names, or teammate names.
- Prefer generic categories over company-specific tooling: CI/CD, observability, issue tracking, code hosting, analytics warehouse.
- Keep examples illustrative, not copied from private code or data.

## Maintenance

- Add a new sub-skill when a pattern repeats at least twice.
- Keep skills short enough to load just-in-time.
- Retire skills that no longer match how Nicole works.
- When a skill starts accumulating domain facts, split those facts into a resource file and keep the skill focused on behavior.
