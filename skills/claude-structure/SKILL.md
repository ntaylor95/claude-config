# Claude Code Structure — Knowledge Base

## Purpose
This skill defines the hierarchy of Claude Code configuration and provides a decision framework for where things should go. Read this before making any classification decisions in the claude-architect agent.

---

## How Claude Code Boots Up

When a Claude Code session starts, it automatically loads in this order:

1. **Global `~/.claude/CLAUDE.md`** — personal cross-repo defaults, always loaded
2. **Global `~/.claude/rules/`** — global modular rules, always loaded
3. **Repo `CLAUDE.md`** — project-specific context, always loaded (cascades through subdirectories)
4. **Repo `.claude/rules/`** — project-specific modular rules, always loaded

Everything else (skills, agents, commands) is **invoked on demand** — Claude does not automatically read or use them unless told to.

---

## The Hierarchy

### 1. CLAUDE.md (Always-On Memory)
- Auto-loaded every session
- Best for: project-specific facts, architecture context, component conventions, team standards
- Should be **lean and scannable** — use resource file references for long content
- Repo `CLAUDE.md` lives at the project root or cascades through subdirectories
- Global `~/.claude/CLAUDE.md` applies across all repos

**What belongs here:**
- Component usage conventions (e.g. "Use FlexRow's `gap` prop, never set gap in .less")
- Known architectural patterns specific to this codebase
- Pointers to available global agents and skills
- Personal preferences (global only)

---

### 2. Rules (`.claude/rules/` or `~/.claude/rules/`)
- Auto-loaded every session alongside CLAUDE.md
- Modular and composable — each rule is a separate file
- Best for: specific, enforceable coding constraints that benefit from being isolated
- Can be repo-specific or global
- Can reference resource files for detail

**What belongs here:**
- Linting-style constraints ("Never use `any` in TypeScript")
- Framework-specific guardrails ("Always use our design system components, not raw HTML elements")
- Security or compliance rules

**CLAUDE.md vs Rules:**
- CLAUDE.md = everything Claude needs to know about working in this repo
- Rules = individual composable guardrails, easier to share and toggle

---

### 3. Skills (`.claude/skills/` or `~/.claude/skills/`)
- **NOT auto-loaded** — invoked on demand
- Each skill lives in its own folder with a `SKILL.md` file
- Best for: repeatable, multi-step processes that are too complex for a rule
- Can reference resource files, examples, templates
- Claude reads the SKILL.md when invoked, loading the expertise just-in-time

**What belongs here:**
- "How to write a unit test for this codebase"
- "How to build an MCP server"
- "How to generate a README"
- "How to review unit tests against our standards"

**Rule vs Skill:**
- Rule = passive standard Claude always follows
- Skill = active process Claude follows when asked

---

### 4. Agents (`.claude/agents/` or `~/.claude/agents/`)
- **NOT auto-loaded** — invoked on demand
- Defined as markdown files with a frontmatter header
- Best for: complex workflows requiring decision-making, routing, or multiple steps
- Can spawn sub-agents using the Task tool
- Can call skills, work from rules, or work from direct prompts
- Do NOT need to call skills — sometimes a focused task prompt is enough

**What belongs here:**
- Orchestrators that route to sub-agents based on user input
- Multi-step review pipelines (write → review → revise)
- Workflows that need parallel execution

**Skill vs Agent:**
- Skill = a playbook (how to do X)
- Agent = a decision-maker (figure out what to do, then do it)

**Two agent patterns:**
- **Sub-agents via Task tool** — Claude orchestrates internally, no terminal needed
- **Separate terminal instances** — human orchestrates, good for large independent workstreams

---

### 5. Commands (`.claude/commands/` or `~/.claude/commands/`)
- Slash commands that invoke skills or agents
- Make skills and agents easily discoverable and repeatable
- Reduces the need to type verbose invocations
- Can be repo-specific or global

**When to create a command:**
- Any skill or agent you find yourself invoking repeatedly
- When you want a short, memorable trigger (e.g. `/unit-test`, `/architect`)

---

### 6. Resource Files
- Reference documents pointed to from CLAUDE.md, rules, or skills
- Keep primary files lean — offload long content to resource files
- Claude reads them on demand when context requires it
- Examples: architecture docs, API references, style guides, example files, templates

---

## Scope: Repo vs Global

| Lives in repo `.claude/` | Lives in `~/.claude/` |
|--------------------------|------------------------|
| Project-specific conventions | Personal cross-repo preferences |
| Codebase-specific skills | General-purpose skills (MCP builder, README generator) |
| Team rules for this project | Rules that apply to all your work |
| Agents for this workflow | Agents that work across projects |

**Default to repo-specific** if unsure. Promote to global when you find yourself copying it across repos.

---

## Classification Decision Framework

### Step 1: Standard or Process?
- **Standard** (Claude should always know/follow this) → Rule or CLAUDE.md
- **Process** (Claude should do this when asked) → Skill, Agent, or Command

### Step 2: If Standard — CLAUDE.md or Rule?
- One-liner convention or project fact → CLAUDE.md
- Standalone enforceable constraint worth isolating → Rule

### Step 3: If Process — Skill or Agent?
- Repeatable how-to with clear steps → Skill
- Requires interviewing user, routing, or spawning sub-processes → Agent
- Both? → Agent that calls a Skill

### Step 4: Add a Command?
- Will this be invoked repeatedly? → Yes, add a Command
- Is it a one-off? → Skip the command

### Step 5: Repo or Global?
- Specific to this codebase/team? → Repo
- Useful across all your projects? → Global

---

## Example Mappings

| What the user wants | Where it goes |
|---------------------|---------------|
| "Use FlexRow's gap prop, not .less" | CLAUDE.md (or Rule if many component conventions) |
| "Unit test naming conventions" | Rule |
| "Write unit tests following our standards" | Skill + Command |
| "Review unit tests and improve them" | Skill + Command |
| "Full unit test pipeline: write, review, revise" | Agent (orchestrator) + sub-agents + Command |
| "Generate a README for any repo" | Global Skill + Global Command |
| "Help me figure out where to put things in Claude" | Global Agent (orchestrator) |

---

## File Structure Reference

```
~/.claude/                          # Global (cross-repo)
├── CLAUDE.md                       # Global always-on context
├── agents/
│   └── claude-architect.md         # Global orchestrator
├── skills/
│   └── claude-structure/
│       └── SKILL.md                # This file
├── rules/                          # Global rules
└── commands/
    └── architect.md                # /architect slash command

.claude/                            # Repo-specific
├── CLAUDE.md (or repo root)        # Project context
├── agents/                         # Project agents
├── skills/                         # Project skills
├── rules/                          # Project rules
└── commands/                       # Project commands
```