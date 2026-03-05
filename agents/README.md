# Claude Code Setup Agents

An orchestrator + builder system for extending your Claude Code configuration. Use `claude-architect` as the entry point whenever you want to add a new rule, skill, agent, or slash command.

---

## `claude-architect` — Orchestrator

Invoke via `/architect`.

Guides you through adding new capabilities to your Claude setup. It interviews you to understand what you're trying to accomplish, classifies the request into the right structure (rule, skill, agent, or command), confirms the file plan with you, then delegates to the appropriate builder sub-agent.

**How it works:**
1. Asks what problem you're trying to solve
2. Classifies it: rule / skill / agent / command / CLAUDE.md entry
3. Asks whether scope is repo-specific or global (`~/.claude/`)
4. Shows you exactly what files will be created and where
5. Spawns the appropriate builder sub-agent
6. Reviews the result with you

---

## Builder Sub-Agents

These are called by `claude-architect` — not invoked directly.

| Agent | Purpose |
|-------|---------|
| `agent-builder` | Builds a new agent for complex, multi-step workflows with routing or decision logic |
| `skill-builder` | Builds a skill folder + `SKILL.md` for a repeatable on-demand process |
| `command-builder` | Builds a slash command shortcut that invokes a skill or agent |
| `rule-builder` | Builds a rule file for a coding standard or constraint Claude should always follow |

---

## Quick Reference: What Goes Where

| You want Claude to... | Use |
|-----------------------|-----|
| Always follow a coding standard | Rule |
| Run a repeatable process when asked | Skill + Command |
| Handle a complex workflow with decisions | Agent + optional sub-agents + Command |
| Always know a project-specific fact | CLAUDE.md entry |
