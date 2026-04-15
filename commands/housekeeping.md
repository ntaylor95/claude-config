# Claude Housekeeping

Run a cleanup pass on `~/.claude` to remove stale runtime artifacts.

## Steps

### 1. Dead session cleanup

Read all files in `~/.claude/sessions/`. Each file contains a JSON object with a `pid` field.
For each file, check if the PID is still a running process using `ps -p <pid>`.
Delete any session file whose PID is no longer alive.

### 2. Stale folder cleanup

Delete all files inside each of these folders (not the folders themselves):
- `~/.claude/shell-snapshots/`
- `~/.claude/session-env/`
- `~/.claude/todos/`
- `~/.claude/paste-cache/`

Use `rm -f` on the files. Do not delete subdirectories.

### 3. File history (ask first)

Ask the user: "Do you also want to clear file-history? This removes undo history for past edits."
If yes, delete all files in `~/.claude/file-history/`.

### 4. Summary

After cleanup, report:
- How many session files were removed (dead PIDs) vs kept (live)
- How many files were deleted from each stale folder
- Whether file-history was cleared
