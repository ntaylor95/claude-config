# .claude — AI Orchestration Toolkit

A portable toolkit for D3 AI orchestration. Drop this into any project's `.claude/` directory.

---

## What's Here

```
.claude/
├── README.md                        ← you are here
├── agents/
│   ├── orchestrator.md              ← /orchestrator — D3 entry point, runs full pipeline
│   ├── executor.md                  ← /executor — writes code from spec
│   ├── tester.md                    ← /tester — writes and runs unit tests
│   ├── reviewer.md                  ← /reviewer — code quality, security, standards review
│   └── self-update.md               ← /self-update — audits and stages system improvements
├── skills/
│   ├── spec-writer/
│   │   └── SKILL.md                 ← interviews you and writes executable specs for Opus
│   └── reverse-spec/
│       └── SKILL.md                 ← extracts business rules from any codebase
└── docs/
    ├── d3-architecture.md           ← full architecture reference + design decisions
    └── spec-template.md             ← blank spec template for manual use
```

---

## Quick Start

### Running the full pipeline
Once you have a spec, hand it to the orchestrator:
> `/orchestrator path/to/my-spec.md`

Opus will run executor → tester → reviewer → self-update autonomously. You'll only be pulled in if something stalls or if the self-update agent proposes changes to rules or agent definitions.

### Running individual agents
You can invoke any agent directly:
- `/executor` — write code from a task or spec
- `/tester` — test code that's already been written
- `/reviewer` — review code before merging
- `/self-update` — audit your .claude directory for gaps

### Writing a new spec
Trigger the `spec-writer` skill:
> *"Help me spec out [what you're building]"*

Claude will interview you section by section and produce a downloadable `.md` spec ready to feed to Opus.

### Reverse-engineering existing code
Trigger the `reverse-spec` skill:
> *"Reverse spec this file: [path/to/file]"*
> *"Extract the business rules from [module name]"*
> *"Point reverse-spec at my repo and analyze [folder]"*

Claude will extract business rules, external dependencies, data models, tech debt, and test coverage gaps. Output is a structured `.md` file per module.

### Going from legacy code to new spec
1. Run `reverse-spec` on the legacy file/module → get `reverse-spec_[module].md`
2. Feed that file into `spec-writer` → pre-populated spec interview
3. Use `/plan` to map the refactor strategy
4. Feed final spec to Opus → agents execute

---

## The D3 Pipeline

```
reverse-spec  →  spec-writer  →  /plan  →  Opus orchestrates
                                                    ↓
                                        executor → tester → reviewer
                                                    ↓
                                           self-update agent
                                                    ↓
                                            git branch
                                                    ↓
                                         your sign-off → merge
```

---

## Agent Roster

| Agent | Invoke | What It Does |
|---|---|---|
| **Orchestrator** | `/orchestrator` | D3 entry point — reads spec, runs full pipeline, escalates when blocked |
| **Executor** | `/executor` | Writes code from spec or task |
| **Tester** | `/tester` | Writes and runs unit tests, validates against success criteria |
| **Reviewer** | `/reviewer` | Code quality, security, and standards review |
| **Self-update** | `/self-update` | Audits .claude/ directory, proposes improvements |

---

## Self-Update Rules

| What | Authority |
|---|---|
| Skills | Auto-approve |
| Specs | Auto-approve |
| Rules | **Human sign-off required** |
| Agent definitions | **Human sign-off required** |

All proposed changes to rules and agent definitions are staged in a git branch. Nothing merges without your explicit approval.

---

## Reference Docs

- **`docs/d3-architecture.md`** — Full architecture, capability matrix, spec format, design decisions
- **`docs/spec-template.md`** — Blank spec template if you want to write manually
