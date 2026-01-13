---
description: Cancel a scheduled Claude task
argument-hint: <task-id>
---

# Cancel Command

Cancel a pending scheduled task, removing it from the system scheduler.

## Context

Use this command to stop a task from executing. This is useful when:
- You no longer need the task to run
- You want to reschedule with different parameters
- You made a mistake in the original schedule

## Instructions

### 1. Parse the Task ID

Get the task ID from `$ARGUMENTS`. Support both full UUIDs and partial matches.

```bash
TASK_ID="$1"
SCHEDULER_DIR="$HOME/.claude-scheduler"
```

### 2. Find the Task

If a partial ID is provided, search for matching tasks:

```bash
# Find tasks that start with the provided ID prefix
MATCHES=$(ls "$SCHEDULER_DIR/tasks" | grep "^$TASK_ID")
```

If multiple matches, ask the user to be more specific.
If no matches, inform the user the task wasn't found.

### 3. Verify Task Status

Read the task's config.json and check status:
- If `pending`: proceed with cancellation
- If `running`: warn that task is currently executing
- If `completed` or `cancelled`: inform user it's already done

### 4. Remove from System Scheduler

**On macOS (launchd)**:
```bash
PLIST_FILE="$HOME/Library/LaunchAgents/com.claude-scheduler.$TASK_ID.plist"
if [ -f "$PLIST_FILE" ]; then
    launchctl bootout gui/$(id -u) "$PLIST_FILE" 2>/dev/null || true
    rm -f "$PLIST_FILE"
fi
```

**On Linux (cron)**:
```bash
crontab -l 2>/dev/null | grep -v "claude-scheduler-$TASK_ID" | crontab - 2>/dev/null || true
```

### 5. Update Task Status

Update the task's config.json:
```json
{
  "status": "cancelled",
  "cancelled_at": "<ISO timestamp>"
}
```

### 6. Update the Index

Update `$HOME/.claude-scheduler/index.json` to reflect the cancelled status.

### 7. Optionally Delete Task Files

Ask the user if they want to completely remove the task files:
- If yes: `rm -rf "$SCHEDULER_DIR/tasks/$TASK_ID"`
- If no: keep files for reference

### 8. Confirm Cancellation

```
✅ Task cancelled successfully!

Task ID: <task-id>
Prompt: "<prompt preview>..."
Was scheduled for: <date/time>

The task has been removed from the system scheduler.
Task files retained at: ~/.claude-scheduler/tasks/<task-id>/
```

## Arguments

- `$ARGUMENTS`: The task ID (full or partial) to cancel

## Example Usage

**Cancel by full ID**:
```
/schedule:cancel abc12345-6789-0abc-def0-123456789abc
```

**Cancel by partial ID**:
```
/schedule:cancel abc123
```

## Error Handling

**Task not found**:
```
❌ Task not found: <id>

No task matching "<id>" was found. Use `/schedule:list` to see all tasks.
```

**Multiple matches**:
```
⚠️ Multiple tasks match "<partial-id>":

- abc12345-... : "Check PR status..."
- abc12346-... : "Run backup..."

Please provide a more specific ID.
```

**Already completed**:
```
ℹ️ Task <id> has already completed.

Use `/schedule:logs <id>` to view its output.
```

**Currently running**:
```
⚠️ Task <id> is currently running!

The task started at <time> and is still executing.
Cancellation will not stop the running process, but will prevent future runs.
Proceed anyway? [y/N]
```

## Notes

- Partial IDs work as long as they uniquely identify a task
- Cancelled tasks are marked but not deleted by default (keeps logs)
- Use `--delete` flag to completely remove task files
- Cancelling a running task won't stop the current execution
