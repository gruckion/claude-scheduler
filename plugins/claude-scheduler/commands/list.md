---
description: List all scheduled Claude tasks
argument-hint: [--all]
---

# List Command

Display all pending scheduled tasks, with optional filtering.

## Context

Use this command to see what tasks are scheduled to run, check their status, and get task IDs for cancellation or log viewing.

## Instructions

### 1. Read the Index File

```bash
INDEX_FILE="$HOME/.claude-scheduler/index.json"
```

If the file doesn't exist, inform the user that no tasks are scheduled.

### 2. Parse Arguments

- No arguments: Show only pending tasks
- `--all`: Show all tasks (pending, running, completed, cancelled)

### 3. Read Each Task's Config

For each task in the index, read `$HOME/.claude-scheduler/tasks/<task-id>/config.json` to get the current status.

### 4. Format Output

Display tasks in a table format:

```
Claude Scheduler - Pending Tasks
================================

ID          | Scheduled For        | Prompt                              | Status
------------|----------------------|-------------------------------------|--------
abc123...   | 2026-02-05 10:00 AM  | Check if PR #936 is merged...      | pending
def456...   | 2026-02-07 09:00 AM  | Run the test suite...              | pending

Total: 2 pending tasks

Use `/schedule:logs <id>` to view task output
Use `/schedule:cancel <id>` to cancel a task
```

### 5. Handle Empty State

If no tasks match the filter:
```
No scheduled tasks found.

Use `/schedule "<prompt>" <time>` to schedule a new task.
```

### 6. Show Time Until Execution

For pending tasks, calculate and display relative time:
```
abc123... | Feb 5, 2026 10:00 AM (in 23 days) | Check if PR... | pending
```

## Arguments

- `$ARGUMENTS`: Optional flags
  - `--all`: Include completed and cancelled tasks
  - (no args): Show only pending tasks

## Example Usage

**List pending tasks**:
```
/schedule:list
```

**List all tasks (including completed)**:
```
/schedule:list --all
```

## Output Format

For pending tasks:
```
Task ID: abc12345-6789-...
Prompt: "Check if PR #936 is merged and comment if not"
Scheduled: Feb 5, 2026 at 10:00 AM (in 23 days)
Working Dir: /Users/you/project
Status: pending
```

For completed tasks (with `--all`):
```
Task ID: xyz98765-4321-...
Prompt: "Run daily backup check"
Scheduled: Jan 10, 2026 at 09:00 AM
Completed: Jan 10, 2026 at 09:02 AM
Exit Code: 0
Status: completed
```

## Notes

- Task IDs can be abbreviated when using them with other commands
- The index is automatically updated when tasks complete or are cancelled
- Use `/schedule:logs <id>` to see the full output of a completed task
