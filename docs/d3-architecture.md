# D3 Architecture Reference
*Derived from: AI orchestration design session*

## Bezos Derivatives — Mapped to Claude

| Derivative | What You Do | Claude Setup |
|---|---|---|
| D1 | Prompt Claude to do tasks | You steer every step |
| D2 | Build workflows, agents, rules | Claude handles tasks end-to-end |
| D3 | Write specs, hand to Opus | Opus manages agents, you review outcomes |
| D4 | Design the AI systems | Agents that can rewrite their own rules |
| D5 | New work only AI teams can do | Not yet defined |

**You are building D3. The self-update agent with human sign-off is the boundary between D3 and D4.**

---

## The Full D3 Pipeline

```
reverse-spec          →       spec-writer      →        /plan         →      Opus executes
(code → .md)               (.md → spec)            (spec → strategy)      (strategy → code)
      ↓                                                                            ↓
Captures:                                                                 tester agent validates
- Business rules                                                          reviewer agent approves
- External deps                                                           self-update agent audits
- Tech debt                                                                       ↓
- Data models                                                          git branch (staged changes)
- Test gaps                                                                       ↓
                                                                       your sign-off → merge
```

---

## Agent Roster

| Agent | Role | Triggers |
|---|---|---|
| Opus (orchestrator) | Reads spec, manages all agents, self-evaluates against success criteria | Spec input |
| Executor | Writes code | Task step in spec |
| Tester | Runs unit tests against executor output | After executor |
| Reviewer | Reviews code quality, patterns, standards | After tester passes |
| Self-update | Audits and proposes updates to skills/rules/agents/specs | On schedule + when Opus flags issues + after every spec |
| Reverse-spec | Points at code, extracts business logic and architecture | On demand |

---

## Capability Matrix

| Capability | Skill | Agent Type | Notes |
|---|---|---|---|
| Write code | code | executor | Sandboxed, output must be validated |
| Test code | unit-test | validator | Runs against executor output |
| Review code | code-review | reviewer | Quality, patterns, standards |
| Extract logic | reverse-spec | analyzer | Any language, any scope |
| Write spec | spec-writer | orchestrator | Section-by-section interview |
| Audit system | self-update | auditor | Runs on schedule + on-demand |
| Research | search | researcher | Use for anything needing live data |
| Decide | rules | decision-maker | Defers to Opus for ambiguous cases |

---

## The 7-Section Spec Format

Every spec fed to Opus must have these sections in order:

### 1. Intent
One to two sentences. What does winning look like? Outcome, not steps.
> *"Every inbound lead is enriched, scored, and routed to the right rep within 60 seconds — no human touch required."*

### 2. Context
- What triggers this workflow
- Starting state (what data/inputs exist)
- Constraints (time, cost, compliance, rate limits, available APIs)

### 3. Success Criteria
Measurable conditions Opus uses to self-evaluate. Must be specific enough to pass or fail.
- At least 3 criteria
- Include: speed, quality thresholds, confidence scores, error rates

### 4. Failure Modes
For each failure: trigger condition + fallback action.
- Missing data → fallback source
- Confidence too low → escalate to human
- API timeout → retry with backoff
- Self-update proposes rule change → stage, notify, halt

### 5. Task Decomposition
Each step includes:
- Step name
- What it does (1 sentence)
- Capability needed
- Agent type
- Input / Output contract

### 6. Decision Points
`IF [condition] THEN [action] ELSE [alternative]`
- Escalation thresholds
- Branching logic
- Priority rules

### 7. Handoff Protocol
- Output format between steps
- What triggers the next step
- Where results are stored/logged
- Final output format and destination

---

## Self-Update Agent Rules

### What it can change autonomously:
- Skills
- Specs

### What requires human sign-off (NEVER auto-commit):
- Rules
- Agent definitions

### When it runs:
- After every spec is written
- When Opus flags something as missing or incorrect
- On a schedule (periodic audit)

### Staging mechanism:
- All proposed changes go to a **git branch**
- Human reviews and approves the branch before merge
- If sign-off not received within agreed time → re-notify, leave staged, continue other work
- If a committed change breaks downstream agents → rollback to previous version

---

## Reverse-Spec → Spec-Writer Handoff Map

| Reverse-Spec Output | Maps to Spec Section |
|---|---|
| Business Rules & Logic | Section 3: Success Criteria + Section 6: Decision Points |
| External Dependencies | Section 2: Context (constraints) + Section 4: Failure Modes |
| Data Models | Section 5: Task Decomposition (inputs/outputs) |
| API Contracts | Section 7: Handoff Protocol |
| Tech Debt | Section 4: Failure Modes |
| Refactor Notes | Section 2: Context |

---

## Key Design Decisions

1. **Specs reference capability, not specific agents** — keeps specs durable as agent library evolves. Opus maps capability → agent at runtime.

2. **Success criteria must be measurable** — if Opus can't pass/fail it, it's not a criterion, it's a wish.

3. **Failure modes are what make this D3** — D2 specs tell agents what to do. D3 specs tell agents what to do when things go wrong.

4. **The self-update agent is the D3/D4 boundary** — autonomous changes to rules and agent definitions = D4. You stay at D3 by requiring human sign-off on those changes.

5. **Git branch as staging layer** — proposed changes live in "pending" state. Nothing goes live without your explicit approval.
