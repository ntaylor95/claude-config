---
name: hackathon-mode
description: Use when Nicole is in fast-prototyping mode — hackathons, internal demos, ML/BQML experiments, proof-of-concept builds where shipping a working thing fast matters more than polish.
---

# Hackathon Mode

Fast prototyping. Working > clean.

## What's Different from Default Engineering Mode

| Default mode | Hackathon mode |
|---|---|
| Tests required | Tests optional unless time allows |
| Type safety strict | Type safety pragmatic |
| README first | README later |
| Feature flags + rollout | Demo on a branch is fine |
| Production-grade error handling | Happy-path first |
| Schema modeling | Throw it in a notebook if it works |

## What's the Same

- Code she can stand behind. "AI suggested it" still isn't a defense.
- Honest scope. If it's a hack, it's a hack — don't oversell to demo audiences.
- Document the *idea* even if not the code, so it can graduate later.

## Defaults

- **BigQuery ML** for fast classification proof-of-concepts.
- **Notebook over service** for one-off explorations.
- **Demo script written before the demo.** Even a 3-line one.

## Graduation Path

If a hack should become a real thing:
- Write the README.
- Add tests.
- Move from notebook to module.
- Plan the rollout and the feature flag.
- Add observability (Sentry, Sumo, NR).
