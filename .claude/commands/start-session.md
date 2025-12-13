---
description: "Initialize ArweaveStamp development session with project context"
allowed-tools: ["Read", "Bash(git:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Start Session - ArweaveStamp

## Purpose
Initialize a productive coding session by loading ArweaveStamp project context, checking git status, and understanding current development phase.

## Workflow

1. **Show current git branch and status**
   ```bash
   !git branch --show-current
   !git status --porcelain
   !git log --oneline --decorate -5
   ```

2. **Load project documentation**
   - Read @README.md for project overview
   - Read @AGENTS.md for module boundaries
   - Read @DEVELOPMENT_PLAN.md for current phase status
   - Read @CLAUDE.md for Claude-specific configuration

3. **Assess current state**
   Display to user:
   - Active Branch: [current branch]
   - Uncommitted Changes: [count and types]
   - Recent Work: [last 5 commits]
   - Current Phase: [1-4 based on timeline]
   - Loaded Context: [list of files read]

4. **Prompt for session goals**
   Ask user:
   - What are your primary goals for this session?
   - Which phase modules will you work on? (storage, analysis, integrity, oracle, postal, mail, payment, tracking)
   - Any known blockers from previous sessions?

5. **Create session log** (optional)
   If user wants tracking, create/update SESSION_LOG.md with:
   - Start time
   - Session goals
   - Active branch
   - Context loaded

## Success Criteria
- All key documentation loaded
- Git status displayed
- User goals captured
- Ready to begin work

## Version History
- 1.0: Initial release
