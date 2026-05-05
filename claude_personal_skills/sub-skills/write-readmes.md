---
name: write-readmes
description: Use when generating, reviewing, or improving a README for a code repository. Encodes Nicole's README contract: why the code exists, how to run it, how to test it, how to deploy it, how to observe it, and what future contributors need to know. Keep tooling generic unless project-specific details are provided.
---

# Write READMEs

Nicole's README contract. Every README should answer six questions.

## Required Sections

1. Why does this code exist?
   - What problem does it solve?
   - Who or what uses it?
   - What systems does it depend on or feed into?

2. How do I run it locally?
   - Setup from a clean clone.
   - Required environment variables with placeholder values only.
   - Local services or dependencies.

3. How do I test it?
   - Test commands.
   - Unit, integration, end-to-end, or manual verification paths.
   - Known test data requirements.

4. How do I deploy it?
   - CI/CD process or link if provided.
   - Branching and release conventions.
   - Rollback notes if relevant.

5. How do I observe it in production?
   - Logs, dashboards, alerts, traces, or error reporting.
   - Health checks and expected signals.
   - Runbook links if provided.

6. What else should a new contributor know?
   - Gotchas.
   - Architecture notes.
   - Ownership/contact pattern if appropriate.
   - Related docs.

## Discovery vs Asking

| Discoverable from repo | Ask the human |
|---|---|
| Package manager and run commands | Why the code exists |
| Test framework | Upstream/downstream systems |
| Docker or local service config | Deployment destination if not obvious |
| CI config files | Observability links or dashboard names |
| Existing docs | Operational gotchas |

Do not fabricate links, dashboards, project names, or owners.

## Style

- Concise and practical.
- Use bullets over paragraphs where possible.
- Use code blocks for commands.
- Link to authoritative docs when links are known.
- Prefer placeholders over private values.
