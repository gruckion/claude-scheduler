---
description: Schedule Claude Code to run a prompt at a future time
argument-hint: "<prompt>" <time-specification> [--remind <duration>]
---

# Schedule Command

Schedule Claude Code to execute a prompt at a specified future time. The task will run autonomously using macOS launchd or Linux cron, with optional reminder notifications and clickable completion alerts.

## Context

Use this command when you want Claude to perform a task in the future, such as:
- Checking on PR status after a review period
- Running periodic checks or maintenance tasks
- Reminding yourself about something with context
- Following up on issues after a delay

## Instructions

### 1. Parse the User's Request

Extract components from `$ARGUMENTS`:
- **Prompt**: The quoted text that describes what Claude should do
- **Time specification**: When to run the task
- **--remind <duration>** (optional): Get notified before task runs (e.g., `--remind 10m`, `--remind 1h`)

**Time formats supported**:
- Relative: `in 1 hour`, `in 30 minutes`, `in 2 days`, `in 1 week`, `tomorrow`, `tomorrow at 9am`
- Absolute: `on 2026-02-15`, `on 2026-02-15 at 14:00`, `at 3pm`, `at 15:00`

**Remind duration formats**:
- Minutes: `5m`, `10m`, `30m`
- Hours: `1h`, `2h`
- Days: `1d`

### 2. Generate Task ID and Paths

```bash
TASK_ID=$(uuidgen | tr '[:upper:]' '[:lower:]')
SCHEDULER_DIR="$HOME/.claude-scheduler"
TASK_DIR="$SCHEDULER_DIR/tasks/$TASK_ID"
```

### 3. Create Task Directory Structure

```bash
mkdir -p "$TASK_DIR"
```

### 4. Calculate the Execution Time

Use the `date` command to calculate the exact execution timestamp:

**For relative times**:
```bash
# "in X hours/minutes/days/weeks"
SCHEDULED_DATE=$(date -v +Xh "+%Y-%m-%d %H:%M:%S")  # macOS
SCHEDULED_DATE=$(date -d "+X hours" "+%Y-%m-%d %H:%M:%S")  # Linux
```

**For absolute times**:
```bash
SCHEDULED_DATE="2026-02-15 14:00:00"
```

### 5. Calculate Reminder Time (if --remind specified)

If user specified `--remind <duration>`:

```bash
# Parse remind duration (e.g., "10m" -> 600 seconds)
REMIND_SECONDS=<parsed duration in seconds>

# Calculate reminder time
SCHEDULED_EPOCH=$(date -j -f "%Y-%m-%d %H:%M:%S" "$SCHEDULED_DATE" "+%s" 2>/dev/null || date -d "$SCHEDULED_DATE" "+%s")
REMINDER_EPOCH=$((SCHEDULED_EPOCH - REMIND_SECONDS))
CURRENT_EPOCH=$(date "+%s")

# Only set reminder if it's in the future
if [ $REMINDER_EPOCH -gt $CURRENT_EPOCH ]; then
    REMINDER_DELAY=$((REMINDER_EPOCH - CURRENT_EPOCH))
    HAS_REMINDER=true
else
    HAS_REMINDER=false
    echo "Note: Reminder time has already passed, skipping reminder"
fi
```

### 6. Save Task Configuration

Create `$TASK_DIR/config.json`:
```json
{
  "id": "<task-id>",
  "prompt": "<the prompt to execute>",
  "scheduled_at": "<ISO timestamp when task should run>",
  "created_at": "<ISO timestamp of creation>",
  "working_dir": "<current working directory>",
  "status": "pending",
  "platform": "darwin|linux",
  "remind_before_seconds": <seconds or null>,
  "reminder_scheduled": <true|false>
}
```

### 7. Save the Prompt

Save the full prompt to `$TASK_DIR/prompt.txt` for reference.

### 8. Create the Reminder Script (if --remind specified)

If `HAS_REMINDER` is true, create `$TASK_DIR/remind.sh`:

```bash
#!/bin/bash
# Claude Scheduler Reminder for Task: <task-id>
TASK_ID="<task-id>"
TASK_DIR="$HOME/.claude-scheduler/tasks/$TASK_ID"
PROMPT_PREVIEW="<first 50 chars of prompt>..."

# Send reminder notification
if [[ "$OSTYPE" == "darwin"* ]]; then
    if command -v terminal-notifier &> /dev/null; then
        terminal-notifier \
            -title "Claude Scheduler - Starting Soon" \
            -subtitle "Task will run in <remind_duration>" \
            -message "$PROMPT_PREVIEW" \
            -sound default
    else
        osascript -e "display notification \"$PROMPT_PREVIEW\" with title \"Claude Scheduler\" subtitle \"Task starting in <remind_duration>\""
    fi
elif [[ "$OSTYPE" == "linux"* ]]; then
    if command -v notify-send &> /dev/null; then
        notify-send "Claude Scheduler - Starting Soon" "Task will run in <remind_duration>: $PROMPT_PREVIEW"
    fi
fi
```

Make it executable:
```bash
chmod +x "$TASK_DIR/remind.sh"
```

### 9. Create the Execution Script

Create `$TASK_DIR/run.sh`:

```bash
#!/bin/bash
# Claude Scheduler Task: <task-id>
# Scheduled: <scheduled_at>
# Prompt: <first 50 chars of prompt>...

TASK_ID="<task-id>"
TASK_DIR="$HOME/.claude-scheduler/tasks/$TASK_ID"
WORKING_DIR="<working_dir>"
PROMPT_FILE="$TASK_DIR/prompt.txt"
LOG_FILE="$TASK_DIR/output.log"
CONFIG_FILE="$TASK_DIR/config.json"
PROMPT_PREVIEW="<first 80 chars of prompt>..."

# Change to the original working directory
cd "$WORKING_DIR" || exit 1

# Mark as running
if command -v jq &> /dev/null; then
    jq '.status = "running" | .started_at = "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
fi

# Log header
echo "=== Claude Scheduler Task Started ===" >> "$LOG_FILE"
echo "Task ID: $TASK_ID" >> "$LOG_FILE"
echo "Time: $(date)" >> "$LOG_FILE"
echo "Working Directory: $WORKING_DIR" >> "$LOG_FILE"
echo "===================================" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"

# Run Claude with JSON output to capture session ID
CLAUDE_OUTPUT=$(claude -p "$(cat "$PROMPT_FILE")" --output-format json 2>&1)
EXIT_CODE=$?

# Extract session ID from JSON output
SESSION_ID=""
if command -v jq &> /dev/null; then
    SESSION_ID=$(echo "$CLAUDE_OUTPUT" | jq -r '.session_id // empty' 2>/dev/null)
fi

# Save session ID for resume functionality
if [ -n "$SESSION_ID" ]; then
    echo "$SESSION_ID" > "$TASK_DIR/session_id.txt"
fi

# Extract and log the result text
RESULT_TEXT=$(echo "$CLAUDE_OUTPUT" | jq -r '.result // empty' 2>/dev/null)
if [ -n "$RESULT_TEXT" ]; then
    echo "$RESULT_TEXT" >> "$LOG_FILE"
else
    # Fallback: log raw output if JSON parsing fails
    echo "$CLAUDE_OUTPUT" >> "$LOG_FILE"
fi

echo "" >> "$LOG_FILE"
echo "=== Task Completed ===" >> "$LOG_FILE"
echo "Exit Code: $EXIT_CODE" >> "$LOG_FILE"
echo "Session ID: $SESSION_ID" >> "$LOG_FILE"
echo "Time: $(date)" >> "$LOG_FILE"

# Mark as completed and save session ID
if command -v jq &> /dev/null; then
    jq '.status = "completed" | .completed_at = "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'" | .exit_code = '"$EXIT_CODE"' | .session_id = "'"$SESSION_ID"'"' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
fi

# Create resume script for click-to-continue functionality
if [ -n "$SESSION_ID" ]; then
    cat > "$TASK_DIR/resume.sh" << RESUME_EOF
#!/bin/bash
# Resume Claude session: $SESSION_ID
osascript -e 'tell application "Terminal" to activate'
osascript -e 'tell application "Terminal" to do script "cd \"$WORKING_DIR\" && claude --resume $SESSION_ID"'
RESUME_EOF
    chmod +x "$TASK_DIR/resume.sh"
fi

# Send completion notification with click-to-resume (macOS)
if [[ "$OSTYPE" == "darwin"* ]]; then
    if command -v terminal-notifier &> /dev/null && [ -n "$SESSION_ID" ]; then
        # Clickable notification that opens Terminal with resumed session
        terminal-notifier \
            -title "Claude Scheduler - Task Completed" \
            -subtitle "Click to continue conversation" \
            -message "$PROMPT_PREVIEW" \
            -execute "$TASK_DIR/resume.sh" \
            -sound default
    else
        # Fallback to basic notification
        osascript -e "display notification \"$PROMPT_PREVIEW\" with title \"Claude Scheduler\" subtitle \"Task completed\""
    fi
fi

# Send notification (Linux)
if [[ "$OSTYPE" == "linux"* ]]; then
    if command -v notify-send &> /dev/null; then
        notify-send "Claude Scheduler - Task Completed" "$PROMPT_PREVIEW"
    fi
fi

# Clean up LaunchAgent (macOS) - self-destruct
if [[ "$OSTYPE" == "darwin"* ]]; then
    PLIST_FILE="$HOME/Library/LaunchAgents/com.claude-scheduler.$TASK_ID.plist"
    if [ -f "$PLIST_FILE" ]; then
        launchctl bootout gui/$(id -u) "$PLIST_FILE" 2>/dev/null || true
        rm -f "$PLIST_FILE"
    fi
    # Also clean up reminder plist if exists
    REMINDER_PLIST="$HOME/Library/LaunchAgents/com.claude-scheduler.$TASK_ID.reminder.plist"
    if [ -f "$REMINDER_PLIST" ]; then
        launchctl bootout gui/$(id -u) "$REMINDER_PLIST" 2>/dev/null || true
        rm -f "$REMINDER_PLIST"
    fi
fi

# Clean up cron entry (Linux)
if [[ "$OSTYPE" == "linux"* ]]; then
    crontab -l 2>/dev/null | grep -v "claude-scheduler-$TASK_ID" | crontab - 2>/dev/null || true
fi

exit $EXIT_CODE
```

Make the script executable:
```bash
chmod +x "$TASK_DIR/run.sh"
```

### 10. Register with System Scheduler

**CRITICAL: Use ONLY ONE scheduling method - choose based on delay time**

First, calculate the delay in seconds until the scheduled time:

```bash
# Calculate seconds until scheduled time
SCHEDULED_EPOCH=$(date -j -f "%Y-%m-%d %H:%M:%S" "$SCHEDULED_DATE" "+%s" 2>/dev/null || date -d "$SCHEDULED_DATE" "+%s")
CURRENT_EPOCH=$(date "+%s")
DELAY_SECONDS=$((SCHEDULED_EPOCH - CURRENT_EPOCH))
echo "Delay until execution: $DELAY_SECONDS seconds"
```

**CHOOSE ONE:**

---

**OPTION A: For near-future tasks (delay < 300 seconds / 5 minutes):**

DO NOT create a launchd plist. Use ONLY a background process with sleep:

```bash
# Schedule reminder if configured (and reminder time is in future)
if [ "$HAS_REMINDER" = true ]; then
    nohup bash -c "sleep $REMINDER_DELAY && $TASK_DIR/remind.sh" > "$TASK_DIR/reminder-background.log" 2>&1 &
    REMINDER_PID=$!
    echo $REMINDER_PID > "$TASK_DIR/reminder.pid"
    echo "Reminder scheduled with PID: $REMINDER_PID"
fi

# Start background process that will run after the delay
nohup bash -c "sleep $DELAY_SECONDS && $TASK_DIR/run.sh" > "$TASK_DIR/background.log" 2>&1 &
BACKGROUND_PID=$!
echo $BACKGROUND_PID > "$TASK_DIR/background.pid"
echo "Started background process with PID: $BACKGROUND_PID"
```

Update config.json with jq:
```bash
jq '.scheduling_method = "background_sleep" | .background_pid = "'$BACKGROUND_PID'"' "$TASK_DIR/config.json" > "$TASK_DIR/config.json.tmp" && mv "$TASK_DIR/config.json.tmp" "$TASK_DIR/config.json"
```

**STOP HERE for near-future tasks. Do not proceed to Option B.**

---

**OPTION B: For tasks MORE than 5 minutes in the future:**

**Schedule reminder first (if configured):**

If `HAS_REMINDER` is true and on macOS, create a reminder LaunchAgent:

Create `~/Library/LaunchAgents/com.claude-scheduler.<task-id>.reminder.plist` with `StartCalendarInterval` set to the reminder time, running `$TASK_DIR/remind.sh`.

**On macOS (launchd) - Main Task:**

Create `~/Library/LaunchAgents/com.claude-scheduler.<task-id>.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude-scheduler.<task-id></string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>$HOME/.claude-scheduler/tasks/<task-id>/run.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Year</key>
        <integer>YYYY</integer>
        <key>Month</key>
        <integer>MM</integer>
        <key>Day</key>
        <integer>DD</integer>
        <key>Hour</key>
        <integer>HH</integer>
        <key>Minute</key>
        <integer>MM</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>$HOME/.claude-scheduler/tasks/<task-id>/launchd-stdout.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/.claude-scheduler/tasks/<task-id>/launchd-stderr.log</string>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
```

Then load it:
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-scheduler.<task-id>.plist
```

Update config.json:
```json
{
  "scheduling_method": "launchd"
}
```

**On Linux (cron)**:

Add a cron entry:
```bash
# Format: minute hour day month * /path/to/run.sh # claude-scheduler-<task-id>
(crontab -l 2>/dev/null; echo "MM HH DD MM * $HOME/.claude-scheduler/tasks/<task-id>/run.sh # claude-scheduler-<task-id>") | crontab -
```

If reminder is configured, also add a cron entry for the reminder script.

Update config.json:
```json
{
  "scheduling_method": "cron"
}
```

### 11. Update the Index

Update `$HOME/.claude-scheduler/index.json` with the new task:
```json
{
  "tasks": [
    {
      "id": "<task-id>",
      "prompt_preview": "<first 100 chars>...",
      "scheduled_at": "<ISO timestamp>",
      "created_at": "<ISO timestamp>",
      "working_dir": "<path>",
      "status": "pending",
      "has_reminder": true|false,
      "remind_before": "<duration string or null>"
    }
  ]
}
```

### 12. Confirm to User

Display a summary:
```
✅ Task scheduled successfully!

Task ID: <task-id>
Prompt: "<prompt preview>..."
Scheduled for: <human-readable date/time>
Working directory: <path>
Reminder: <X minutes/hours before> (if configured)

Features:
- 🔔 You'll receive a notification when the task completes
- 🖱️ Click the notification to resume the conversation in Terminal
- ⏰ Reminder notification <X> before execution (if configured)

Commands:
- `/schedule:list` - See all scheduled tasks
- `/schedule:logs <task-id>` - View output after execution
- `/schedule:cancel <task-id>` - Cancel this task
```

## Arguments

- `$ARGUMENTS`: The full argument string containing:
  - Quoted prompt (what Claude should do)
  - Time specification (when to run it)
  - `--remind <duration>` (optional): Get notified before task runs

## Example Usage

**Basic scheduling**:
```
/schedule "Check if PR #936 is merged and comment if not" in 23 days
```

**With reminder**:
```
/schedule "Run the test suite and report any failures" tomorrow at 9am --remind 10m
```

**Specific date with reminder**:
```
/schedule "Review the deployment status" on 2026-02-15 at 14:00 --remind 1h
```

**Short delay**:
```
/schedule "Check CI status" in 2 hours --remind 15m
```

## Notes

- Tasks are stored in `~/.claude-scheduler/tasks/<task-id>/`
- On macOS, tasks use LaunchAgents which auto-clean after execution
- On Linux, tasks use cron entries which are removed after execution
- The working directory is preserved - Claude runs from where you scheduled it
- Output is captured to `output.log` in the task directory
- Session ID is saved for click-to-resume functionality
- Tasks self-destruct after running (script stays for logs, scheduler entry removed)

## Dependencies

**Required:**
- `jq` - For JSON parsing (session ID extraction)

**Optional (enhances experience):**
- `terminal-notifier` - Clickable notifications on macOS (install: `brew install terminal-notifier`)
- `notify-send` - Notifications on Linux

**Fallback:**
- If `terminal-notifier` is not installed, basic `osascript` notifications are used (not clickable)
