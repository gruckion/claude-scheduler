---
description: View logs from a scheduled Claude task
argument-hint: <task-id> [--tail] [--follow]
---

# Logs Command

View the output logs from a scheduled Claude task.

## Context

Use this command to:
- See what Claude did when a scheduled task ran
- Debug issues with failed tasks
- Review the results of completed tasks
- Monitor a currently running task

## Instructions

### 1. Parse Arguments

```bash
TASK_ID="$1"  # Required
# Optional flags: --tail, --follow, --lines N
```

### 2. Find the Task

Locate the task directory, supporting partial ID matching:

```bash
SCHEDULER_DIR="$HOME/.claude-scheduler"
TASK_DIR="$SCHEDULER_DIR/tasks/$TASK_ID"
```

If partial ID, find matches and resolve.

### 3. Check Task Exists

Verify the task directory and log file exist:

```bash
LOG_FILE="$TASK_DIR/output.log"
CONFIG_FILE="$TASK_DIR/config.json"
```

### 4. Read Task Status

Check config.json to understand the task state:
- `pending`: Task hasn't run yet
- `running`: Task is currently executing
- `completed`: Task finished
- `cancelled`: Task was cancelled before running

### 5. Display Task Info Header

Show context about the task:

```
Task: <task-id>
Status: <status>
Prompt: "<prompt>"
Scheduled: <scheduled_at>
Working Dir: <working_dir>
```

For completed tasks, also show:
```
Started: <started_at>
Completed: <completed_at>
Exit Code: <exit_code>
```

### 6. Display Logs

**Default (full log)**:
Read and display the entire `output.log` file.

**With `--tail` or `--lines N`**:
Show only the last N lines (default 50).

**With `--follow`** (for running tasks):
```bash
tail -f "$LOG_FILE"
```

### 7. Handle Missing Logs

If no log file exists:
- For pending tasks: "Task hasn't run yet. Scheduled for: <date>"
- For cancelled tasks: "Task was cancelled before execution."
- For running tasks: "Task is running but no output yet."

## Arguments

- `$1` / `$ARGUMENTS`: The task ID (full or partial)
- `--tail`: Show only the last 50 lines
- `--lines N`: Show the last N lines
- `--follow` / `-f`: Follow the log in real-time (for running tasks)
- `--all`: Include launchd stdout/stderr logs (macOS)

## Example Usage

**View full log**:
```
/schedule:logs abc123
```

**View last 20 lines**:
```
/schedule:logs abc123 --lines 20
```

**Follow a running task**:
```
/schedule:logs abc123 --follow
```

## Output Format

```
═══════════════════════════════════════════════════════════
Task: abc12345-6789-0abc-def0-123456789abc
Status: completed
Prompt: "Check if PR #936 is merged and comment if not"
Scheduled: Feb 5, 2026 at 10:00 AM
Started: Feb 5, 2026 at 10:00:01 AM
Completed: Feb 5, 2026 at 10:02:15 AM
Exit Code: 0
═══════════════════════════════════════════════════════════

=== Claude Scheduler Task Started ===
Task ID: abc12345-6789-0abc-def0-123456789abc
Time: Thu Feb  5 10:00:01 PST 2026
Working Directory: /Users/you/project
===================================

I'll check if PR #936 has been merged...

[Claude's output from the task]

=== Task Completed ===
Exit Code: 0
Time: Thu Feb  5 10:02:15 PST 2026
```

## Error Handling

**Task not found**:
```
❌ Task not found: <id>

Use `/schedule:list --all` to see all tasks.
```

**Pending task**:
```
ℹ️ Task <id> hasn't run yet.

Scheduled for: Feb 5, 2026 at 10:00 AM (in 23 days)
Prompt: "Check if PR #936 is merged..."

The log file will be available after the task executes.
```

**Cancelled task with no logs**:
```
ℹ️ Task <id> was cancelled before execution.

Cancelled at: Jan 13, 2026 at 2:30 PM
No logs available.
```

## Notes

- Logs are stored at `~/.claude-scheduler/tasks/<id>/output.log`
- On macOS, additional launchd logs may be at `launchd-stdout.log` and `launchd-stderr.log`
- Use `--follow` to watch a running task in real-time (Ctrl+C to stop)
- Logs are retained even after task completion for reference
