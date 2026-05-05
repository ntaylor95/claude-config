---
name: hackathon-mode
description: Use when Nicole is fast-prototyping: hackathons, demos, proof-of-concepts, experiments, or exploratory AI/data builds where a working demonstration matters more than production polish.
---

# Hackathon Mode

Fast prototyping. Working beats polished.

## What's Different

| Default mode | Hackathon mode |
|---|---|
| Tests expected | Tests optional unless time allows |
| Type safety strict | Type safety pragmatic |
| README early | README after the demo, unless needed to run it |
| Production rollout planning | Demo branch is fine |
| Robust error handling | Happy path first |
| Clean architecture | Clear enough to explain |

## What's the Same

- Nicole still owns the result.
- Do not oversell a prototype as production-ready.
- Document the idea well enough that it can graduate later.
- Keep demos honest about limitations.

## Defaults

- Start with the smallest working loop.
- Write the demo script before the demo.
- Use synthetic or safe sample data unless real data is explicitly appropriate.
- Capture TODOs as graduation criteria, not shame.

## Graduation Path

If the prototype should become real:

- Write or update the README.
- Add tests.
- Replace notebooks/scripts with maintainable modules where needed.
- Add config, secrets handling, observability, and rollout plan.
- Identify what must be deleted before production.
