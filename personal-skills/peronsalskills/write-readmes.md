---
name: write-readmes
description: Use when generating, reviewing, or improving a README for any code repository. Encodes Nicole's required contract — every README must explain why the code exists, how to run/test/deploy/observe it, with concrete links to TeamCity/Octopus/Argo, Sentry, Sumo, and New Relic.
---

# Write READMEs

Nicole's required README contract. Every README should answer all six questions:

## Required Sections

1. **Why does this code exist?**
   - What problem does it solve?
   - What other systems does it work with (upstream and downstream)?

2. **How do I run it locally?**
   - Setup steps from a clean clone.
   - Required env vars (with placeholder values, never real ones).

3. **How do I test it?**
   - Test command(s).
   - Any test categories (unit / integration / e2e) and how to run each.

4. **How do I deploy it?**
   - Link to the TeamCity build configuration.
   - Link to the Octopus project (or Argo app, depending on stack).
   - Branching/release conventions.

5. **How do I observe it in production?**
   - Sentry project name.
   - Sumo Logic source category.
   - New Relic dashboard link(s).

6. **Anything else a new engineer should know?**
   - Known gotchas, on-call notes, runbook links if any.

## Discovery vs. Asking

When generating a README from a repo:

| Discoverable from repo | Must ask the human |
|---|---|
| Package manager, run/test commands | Why the code exists |
| Dockerfile presence | Upstream/downstream systems |
| CI config files | Sentry project name |
| Test framework | Sumo source category |
| Branch conventions | New Relic dashboard URLs |

If a CI/observability link can't be inferred from URL patterns, ask — don't fabricate.

## Style

- Concise. Bullet points over paragraphs where they fit.
- Code blocks for commands.
- Link, don't paraphrase, when there's an authoritative source elsewhere.
