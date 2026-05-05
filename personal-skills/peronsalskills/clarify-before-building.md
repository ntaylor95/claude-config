---
name: clarify-before-building
description: Use when Nicole gives a task with ambiguous scope, missing inputs, or multiple plausible interpretations. Encodes her preference for surfacing questions upfront rather than guessing and producing wrong output.
---

# Clarify Before Building

Nicole would rather be asked one good question than receive five wrong attempts.

## Things That Always Need Clarifying

- **Date ranges** if not specified. Don't default silently. Ask.
- **Schema** if working with data. Ask for the shape; don't invent column names.
- **Multiple plausible interpretations.** Surface them as A/B/C and ask which.
- **Tech stack assumptions** if the snippet could be Python or TypeScript. Ask.
- **Audience** for written content (internal vs. external; technical vs. non-technical).

## How to Ask

- Lead with what you'll do once you have the answer. ("I can build this — quick question first.")
- Group related questions. Don't ping-pong.
- Cap at 3 questions; rank by what blocks progress.
- Offer best-guess defaults so she can say "yes, that" instead of typing.

## When NOT to Ask

- Trivial conventions that have a clear default (file naming, formatting).
- Things she's stated before in the session.
- Things her domain/workflow context already answers — load those first, then ask only what's still missing.

## Anti-Patterns

- ❌ Hallucinating field names instead of asking.
- ❌ Asking ten questions before doing anything.
- ❌ Asking after producing a wrong attempt.
- ❌ Asking about things her behaviors/workflow files already answer.
