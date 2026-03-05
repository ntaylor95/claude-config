# Spec: [Name]
*Created: [date]*
*Author: [your name]*
*Version: 1.0*

---

## Intent
> What does winning look like? 1-2 sentences. Outcome, not steps.

[Replace this with your intent statement]

---

## Context

**Trigger:** [What starts this workflow — schedule, event, webhook, user action]

**Starting State:**
- [What data/inputs exist at the start]
- [What systems are available]
- [What has already happened before this runs]

**Constraints:**
- Time: [SLA or deadline]
- Cost: [budget or rate limits]
- Compliance: [any legal/regulatory/privacy constraints]
- APIs available: [list key services]

---

## Success Criteria
> Measurable. Opus uses these to self-evaluate. Must be passable or faileable.

1. [Criterion — include threshold, e.g. "Completed within 60 seconds"]
2. [Criterion — include quality measure, e.g. "Confidence score > 0.85"]
3. [Criterion — include error rate, e.g. "Zero PII written to logs"]
4. [Add more as needed]

---

## Failure Modes
> For each failure: trigger condition + fallback action.

| Trigger Condition | Fallback Action |
|---|---|
| Missing required input data | [action] |
| Confidence score below threshold | Escalate to human |
| External API timeout | Retry with exponential backoff, max 3 attempts |
| Self-update proposes rule/agent change | Stage in git branch, notify human, halt until sign-off |
| Committed change breaks downstream agent | Rollback to previous version |
| [Add more] | [action] |

---

## Task Decomposition
> Ordered steps. Each step has a named capability and agent type.

| # | Step Name | What It Does | Capability | Agent Type | Input | Output |
|---|---|---|---|---|---|---|
| 1 | [name] | [1 sentence] | [research/write/analyze/code/decide/retrieve/send] | [executor/reviewer/decision-maker/orchestrator] | [what comes in] | [what goes out] |
| 2 | | | | | | |
| 3 | | | | | | |

*Note: Reference capability, not specific agent. Opus maps capability → agent at runtime.*

---

## Decision Points
> Where Opus makes judgment calls. Logic must be explicit.

- `IF [condition] THEN [action] ELSE [alternative]`
- `IF [condition] THEN [action] ELSE [alternative]`
- `IF confidence < [threshold] THEN escalate to human ELSE proceed`
- `IF [external API] fails THEN use fallback ELSE continue`

---

## Handoff Protocol

**Format between steps:** [JSON / plain text / structured object — describe schema if JSON]

**Trigger for next step:** [what causes each step to hand off]

**Storage / Logging:**
- Results stored at: [location]
- Logs written to: [location]
- PII handling: [scrubbed / not logged / encrypted]

**Final Output:**
- Format: [.md / JSON / database record / email / etc.]
- Destination: [where the final output goes]
- Success notification: [who/what gets notified on completion]

---

*Spec Version 1.0 — Ready for Opus*
