---
description: Show Claude Scheduler help and available commands
argument-hint: [command]
---

# Help Command

Display help information for the Claude Scheduler plugin.

## Instructions

### If No Arguments

Display the overview:

```
Claude Scheduler - Schedule Claude to run tasks at future times

Commands:
  /schedule "<prompt>" <time>   Schedule a new task
  /schedule:list [--all]        List scheduled tasks
  /schedule:cancel <id>         Cancel a pending task
  /schedule:logs <id>           View task output
  /schedule:help [command]      Show this help

Time Formats:
  Relative:  in 30 minutes, in 2 hours, in 3 days, tomorrow, tomorrow at 9am
  Absolute:  on 2026-02-15, on 2026-02-15 at 14:00, at 3pm

Examples:
  /schedule "Check PR status" in 2 days
  /schedule "Run tests" tomorrow at 9am
  /schedule:list
  /schedule:cancel abc123
  /schedule:logs abc123

Storage: ~/.claude-scheduler/
Platforms: macOS (launchd), Linux (cron)
```

### If Command Argument Provided

Show detailed help for that specific command:

- `schedule`: Explain time formats, prompt quoting, working directory preservation
- `list`: Explain --all flag, status meanings
- `cancel`: Explain partial ID matching, what happens to files
- `logs`: Explain --tail, --follow, --lines flags

## Arguments

- `$ARGUMENTS`: Optional command name for detailed help

## Example Usage

```
/schedule:help
/schedule:help schedule
/schedule:help list
```

## Notes

- This is a quick reference; see README.md for full documentation
- Tasks are stored in `~/.claude-scheduler/tasks/`
- macOS uses LaunchAgents, Linux uses cron
