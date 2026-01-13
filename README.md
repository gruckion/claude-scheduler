![Claude Scheduler](assets/banner.png)

# Claude Scheduler

**Schedule Claude to work while you sleep.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> "Check if that PR got merged in 3 days and ping me if not."

One command. Claude runs autonomously at the scheduled time, completes the task, and sends you a clickable notification to continue the conversation.

---

## The Problem

You're deep in a coding session. You need to follow up on something later—a PR review, a deployment check, a CI status. But you'll forget. Or you'll set a calendar reminder that lacks context.

## The Solution

```bash
/schedule "Check PR #42 status and comment if there are issues" in 2 days --remind 1h
```

Claude remembers. Claude executes. Claude notifies you. Click the notification → resume the conversation with full context.

---

## Quick Start

### Install

```bash
# Add the marketplace
/plugin marketplace add gruckion/claude-scheduler

# Install the plugin
/plugin install claude-scheduler
```

### Dependencies

```bash
brew install terminal-notifier  # Clickable notifications (macOS)
```

### Schedule Your First Task

```bash
/schedule "Run tests and summarize failures" tomorrow at 9am --remind 10m
```

Done.

---

## What You Get

| Feature | Description |
|---------|-------------|
| **Fire-and-forget scheduling** | Set it and forget it. Claude runs on time. |
| **Reminder alerts** | Get notified before tasks run (`--remind 10m`) |
| **Click-to-resume** | Notification → Terminal → Continue the conversation |
| **Full context** | Claude remembers the working directory, the prompt, everything |
| **Cross-platform** | macOS (launchd) and Linux (cron) |

---

## Commands

```bash
/schedule "<prompt>" <when> [--remind <duration>]   # Schedule a task
/schedule:list [--all]                               # List tasks
/schedule:cancel <task-id>                           # Cancel a task
/schedule:logs <task-id> [--follow]                  # View output
/schedule:help                                       # Quick reference
```

---

## Time Formats

**Relative:**
```bash
in 30 minutes
in 2 hours
in 3 days
tomorrow
tomorrow at 9am
```

**Absolute:**
```bash
at 3pm
at 15:00
on 2026-02-15
on 2026-02-15 at 14:00
```

**Reminders:**
```bash
--remind 5m    # 5 minutes before
--remind 1h    # 1 hour before
--remind 1d    # 1 day before
```

---

## Real Examples

### PR Follow-up
```bash
/schedule "Check if PR #123 is merged. If yes, celebrate. If no, summarize review comments." in 2 days --remind 1h
```

### Standup Prep
```bash
/schedule "Prepare standup: git log from yesterday, open PRs, TODO comments" tomorrow at 8:45am --remind 15m
```

### Deployment Monitor
```bash
/schedule "Check production logs for errors since last deploy" in 4 hours --remind 30m
```

### CI Status
```bash
/schedule "Check if CI passed and report any failures" in 20 minutes
```

---

## How It Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   You schedule  │ ──▶ │  System waits   │ ──▶ │  Claude runs    │
│   a task        │     │  (launchd/cron) │     │  autonomously   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Continue the  │ ◀── │  Click the      │ ◀── │  Notification   │
│   conversation  │     │  notification   │     │  appears        │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

1. **Schedule** → Task saved to `~/.claude-scheduler/tasks/`
2. **Wait** → System scheduler handles timing (launchd on macOS, cron on Linux)
3. **Execute** → Claude runs with your prompt in the original directory
4. **Notify** → Clickable notification with task summary
5. **Resume** → Click to open Terminal with `claude --resume`

---

## Platform Support

| Platform | Scheduler | Notifications | Click-to-Resume |
|----------|-----------|---------------|-----------------|
| macOS | launchd | terminal-notifier / osascript | Yes |
| Linux | cron | notify-send | No |

---

## Directory Structure

```
~/.claude-scheduler/
├── tasks/
│   └── <task-id>/
│       ├── config.json       # Task metadata + session ID
│       ├── prompt.txt        # The full prompt
│       ├── run.sh            # Execution script
│       ├── remind.sh         # Reminder script (if --remind)
│       ├── resume.sh         # Click-to-resume script
│       ├── output.log        # Claude's output
│       └── session_id.txt    # Session ID for resume
└── index.json                # Task index
```

---

## Troubleshooting

**Task didn't run?**
- Check `~/.claude-scheduler/tasks/<id>/launchd-stderr.log`

**Notification not clickable?**
- Install: `brew install terminal-notifier`

**Resume not working?**
```bash
claude --resume $(cat ~/.claude-scheduler/tasks/<id>/session_id.txt)
```

---

## Manual Installation

```bash
git clone https://github.com/gruckion/claude-scheduler
cd claude-scheduler
claude --plugin-dir ./plugins/claude-scheduler
```

---

## License

MIT

---

**Stop forgetting. Start scheduling.**
