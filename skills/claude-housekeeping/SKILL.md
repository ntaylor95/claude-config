# Claude Housekeeping — Folder Reference

A reference guide to what each runtime folder in `~/.claude` holds, how it grows, and when it's safe to clean up.

---

## Folder Summary

| Folder | Purpose | Auto-cleaned? | Safe to clear? |
|---|---|---|---|
| `sessions/` | Active session tracker (PID + working dir per Claude window) | On clean exit | Dead PIDs only |
| `session-env/` | Env var snapshots per session UUID | No | Yes — all |
| `shell-snapshots/` | Shell state captures (aliases, functions, env) taken before Bash runs | No | Yes — all |
| `todos/` | Task/todo state files from agent runs via TodoWrite | No | Yes — all |
| `file-history/` | Undo history for file edits | No | Yes, if no recent edits to undo |
| `paste-cache/` | Temporary storage for large pasted content | No | Yes — all |
| `projects/` | Per-repo context and memory used across sessions | No | Do not delete |
| `tasks/` | Background task tracking (TaskCreate/TaskUpdate) | No | After confirmed complete |
| `plans/` | Plan mode artifacts | No | After plans are done |
| `debug/` | Debug logs | No | Yes |
| `cache/` | General internal cache | No | Yes |
| `plugins/` | Installed plugins | No | Do not delete |
| `agents/` | Your configured agents | No | Do not delete |
| `skills/` | Your configured skills | No | Do not delete |
| `history.jsonl` | Full conversation history | No | Only if you want to wipe history |
| `statsig/` | Feature flag / analytics cache | No | Yes |

---

## What accumulates fastest

- **`shell-snapshots/`** — Created on every Bash tool call. Grows quickly during active sessions.
- **`todos/`** — Created by every agent run that uses TodoWrite. Never auto-removed.
- **`session-env/`** — One entry per session UUID. Orphaned when sessions exit abruptly.
- **`paste-cache/`** — Grows with large pastes. Not bounded.

---

## Cleanup command

Run `/housekeeping` to interactively clean up stale artifacts.
It will:
1. Remove session files for dead PIDs
2. Clear `shell-snapshots/`, `session-env/`, `todos/`, `paste-cache/`
3. Optionally clear `file-history/`
